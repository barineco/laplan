# IR Pipeline Walkthrough

This document picks a single concrete function from resolver.lex and traces every stage from Lex₀ to final output, then follows the return path through inverse. The role of each layer is covered in general terms in [ir.md](ir.md); this document is a step-by-step guide for reading how a function descends through the layers into language code and comes back.

## Main: checkNeeds

`checkNeeds` is the shortest of the 9 runtime resolver functions. Its body is a single line, and the roundtrip succeeds across all 21 languages, making it possible to observe the layer correspondences without distortion.

### Lex₀: KDL declaration in resolver.lex

`axiom/resolver.lex` contains `checkNeeds` as follows:

```kdl
fn "checkNeeds" {
  param "goal" type="GoalSpec"
  return-type "CheckNeedsOutput"
  app "lang.checkNeedsForGoal" {
    $local.goal
  }
}
```

One parameter, one return type, and the body is a single function application `lang.checkNeedsForGoal($local.goal)`. `$local.goal` is syntactic sugar for a `var "local.goal"` node, corresponding to the `Var` variant of FnExpr. This is the shallowest representation; after passing through the KDL parser into `parse_resolver_lex`, it enters the Lex₁ layer.

### Lex₁: FnDef

`parse_resolver_lex` transforms KDL into `Vec<FnDef>`. The value corresponding to `checkNeeds` is roughly:

```
FnDef {
  name: "checkNeeds",
  params: [("goal", "GoalSpec")],
  return_type: "CheckNeedsOutput",
  body: Algebraic(
    FnExpr::App("lang.checkNeedsForGoal", [FnExpr::Var("local.goal")]),
  ),
}
```

The body is held as `FnBody::Algebraic(FnExpr)`, with `FnExpr::App` representing function application. NSID prefixes like `local.` / `lang.` are retained at the Lex₁ stage and stripped according to each language's naming conventions during emit.

### Lex₂: RuntimeFn

For languages that go through Lex₂ (rust, python, etc.), `lowering::fn_to_lex2` lowers the Lex₁ `FnExpr` to `RuntimeFn`. A single App expression like `checkNeeds` becomes a `Stmt` sequence:

```
RuntimeFn {
  name: "checkNeeds",
  params: [("goal", "GoalSpec")],
  return_type: "CheckNeedsOutput",
  body: [
    Stmt::Return(
      Expr::Call("checkNeedsForGoal", [Expr::Var("goal")]),
    ),
  ],
}
```

The body becomes `Vec<Stmt>`, closed by `Stmt::Return` at the tail. NSID prefixes are prepared for stripping during emit.

### Haskell output (direct Lex₁ emit)

Haskell has a `functional {}` section, so it emits directly from Lex₁. `template_engine_fn::fn_template_emit_runtime` handles this.

```haskell
checkNeeds :: GoalSpec -> CheckNeedsOutput
checkNeeds goal =
  checkNeedsForGoal goal
```

`FnExpr::App` maps directly to Haskell's space application. By not going through Lex₂, the functional expression tree is preserved almost intact in the output.

### Rust output (via Lex₂, brace-based)

Rust is lowered to Lex₂ and then emitted via `template_engine::template_emit_fn`. The following is the raw emit output with tab indentation (before rustfmt formatting).

```rust
pub fn checkNeeds(goal: GoalSpec) -> CheckNeedsOutput {
	return checkNeedsForGoal(goal);
}
```

The `Vec<Stmt>` body is expanded into a brace block, with `Stmt::Return` becoming `return ...;`. For a single App expression, the shape remains straightforward whether starting from Lex₁ or Lex₂.

### Python output (via Lex₂, indent-based)

Python also goes through Lex₂, but block boundaries are indicated by indentation.

```python
def checkNeeds(goal: GoalSpec):
    return checkNeedsForGoal(goal)
```

The shape is isomorphic to Rust, but the `def` declaration and indent handling are determined by the `fn-open` / `indent` settings in mapping.lex. For the inverse side to detect block endings, Python's mapping.lex has a `«dedent»` directive (see [mapping-template.md](../reference/mapping-template.md) for details).

### Inverse: return path

When running inverse on the Rust output as source, the layers are traversed in reverse order:

1. The pattern matcher in `ast_inverse::engine` scans the entire `pub fn checkNeeds(...) { ... }`, returning a `MatchResult` with `consumed` byte count and captures
2. A `RuntimeFn` is assembled from captures, extracting the body as `Vec<Stmt>`
3. `reverse_lower::reverse_lower` restores the entire body (a single `Stmt::Return`) to `FnBody::Algebraic(FnExpr::App(...))`
4. The resulting `FnDef` matches the original Lex₁ representation

`checkNeeds` is a representative case where this roundtrip succeeds across all 21 languages. No residuals occur at any stage, and no fallback to `Stmt::Raw` / `Expr::Raw` happens.

## Supplement: classifyCandidate

`classifyCandidate` is a function at `axiom/resolver.lex:50-138` with nested case branches, filters, and let-in expressions. A full trace of all stages would exceed the scope of this document, so here we use the Rust output as material to point out "where the Lex₁ representation cannot be fully recovered."

### Structures that break during lowering

The Lex₁ body is a single expression combining case clauses, filters, and let-in, but lowering to Lex₂ fragments it:

- Case clauses are expanded into if-chains
- Filters are expanded into `for` + `if` + mutable list operations
- Each let-in binding becomes a `Stmt::Let` / `Stmt::Mut` sequence

This is the information intentionally discarded on the forward side. The Rust emit output looks like this (excerpt):

```rust
pub fn classifyCandidate(marking: Marking, recipes: HashMap<String, RecipeEntry>,
    requirements: HashMap<String, RecipeRequirements>,
    acc: (Vec<SolveCandidate>, Option<BoundaryKind>), id: String)
    -> (Vec<SolveCandidate>, Option<BoundaryKind>) {
	let candidates = (acc).0;
	let lastBoundary = (acc).1;
	let requirement = requirements[id.as_str()].clone();
	let mut missingCap = vec![].to_owned();
	for capability in &requirement.requires_capability {
		if !(marking.contains_key(capability.as_str())) {
			missingCap = { let mut v = missingCap; v.push(capability.clone()); v };
		}
	}
	if !(missingCap.runtime_is_empty()) {
		return (candidates, Just(Some(BoundaryKind::Capability(missingCap))));
	} else {
		// ... (further if-else branching on recipe presence)
	}
}
```

`$local.lastBoundary` becomes `lastBoundary` in Rust output, and the list emptiness check becomes a `runtime_is_empty` method call. Case and filter are both expanded into `for` + `if` chains, and let-in bindings are opened into `let` / `let mut` sequences.

### Points likely to become residuals in inverse

When this source is run through inverse, pattern matching succeeds up to `Vec<Stmt>`, but `reverse_lower` cannot restore parts to Lex₁ form. Typical candidates are:

- `{ let mut v = missingCap; v.push(capability.clone()); v }` block expressions (cannot be collapsed into Lex₁'s `Concat`; retained as Lex₂ Stmt sequences)
- `.as_str()` and `.clone()` method chains in `requirements[id.as_str()].clone()` (no `MethodChain` variant on the Lex₁ side; without an appropriate axis to fold into App, they remain as Lex₂)
- Nested Construct in `Just(Some(BoundaryKind::Capability(missingCap)))` (expressible in principle via Lex₁'s `Construct`, but can fail due to abbreviations or type elisions on the source side)

Expressions matching these cases fall into `None`-returning match arms in `lift_expr` of `compiler/ir/src/reverse_lower.rs`, and the calling `recognize_*` functions give up on Algebraic-izing the entire body, choosing `FnBody::Procedural(Vec<Stmt>)` instead. As a result, this function is retained as Lex₂, not Lex₁, at the inverse pipeline's exit.

### Summary

To fully restore `classifyCandidate` to Lex₁ across all 21 languages, two things must align: the pattern match stage needs to structurally capture each language's block expressions / method chains / nested Constructs, and the lift stage needs arms to accept `MethodChain` and similar forms. If either is missing, residuals remain at Lex₂. This document stops here, using the above 3 points as landmarks.

## Related docs

- IR layer overview: [ir.md](ir.md)
- Lex layer classification: [../reference/layers.md](../reference/layers.md)
- mapping.lex template syntax: [../reference/mapping-template.md](../reference/mapping-template.md)
- Forward emit implementation: [synthesis.md](synthesis.md)
