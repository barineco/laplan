# Intermediate Representation

`laplan-ir` provides the normalized representation of `.lex` declarations and handles expressions and statements through a two-layer IR: Lex₁ (functional) and Lex₂ (imperative).

## IR Hierarchy Overview

Between the `.lex` source and the final output, the pipeline passes through five intermediate representation stages. Each stage discards specific information to produce a form suited to the next stage.

```mermaid
flowchart TD
    src[".lex / .json<br/>source code"]
    kdl["KDL Document<br/>(raw parse result)"]
    ast["AST<br/>(KDL node tree)"]
    ir["Normalized IR<br/>(RuleBundle, LexiconIr,<br/>Module, Mapping, ...)"]
    lex1["Lex₁: FnExpr<br/>(functional IR)"]
    lex2["Lex₂: Stmt / Expr<br/>(imperative IR)"]
    tt["TransitionTable<br/>(Petri net encoding)"]

    src -->|"laplan-kdl<br/>lexer + parser"| kdl
    kdl -->|"kdl_to_lex<br/>syntactic sugar expansion"| ast
    ast -->|"elaborate<br/>normalization + dependency resolution"| ir
    ir -->|"parse_resolver_lex<br/>rule body → expression"| lex1
    lex1 -->|"fn_to_lex2 (lowering)<br/>ANF transformation"| lex2
    ir -->|"morphisms_to_transitions<br/>rule → arcs"| tt

    lex1 -->|"template_engine_fn<br/>Haskell, OCaml, ..."| out1["Lex₁ language output"]
    lex2 -->|"template_engine<br/>Rust, TS, Python, ..."| out2["Lex₂ language output"]
    lex2 -->|"wasm_emit<br/>direct code generation"| out3[".wasm binary"]
    tt -->|"solver (BFS)"| out4["reachable paths"]
```

The Lex₁ → Lex₂ lowering and the IR → TransitionTable conversion are independent paths. From the same normalized IR, two different pieces of information are extracted for two different purposes: code generation and reachability analysis.

### What Each Stage Discards and Gains

| Stage | Representation | Discards | Gains |
|---|---|---|---|
| KDL Document | neco-kdl generic KDL nodes | - | Grammar correctness guarantee |
| AST | `.lex` semantic nodes | KDL syntax redundancy (`{`, `}`, `;`) | `.lex`-specific structure (rule, morph, family, etc.) |
| Normalized IR | RuleBundle, LexiconIr, etc. | Shorthand forms, syntactic sugar | Type connections, NSID resolution, normalized dependencies |
| Lex₁ (FnExpr) | Functional expression tree | Declaration structure | Pure expression semantics (let-in, fold, recurse) |
| Lex₂ (Stmt/Expr) | Imperative statement sequence | Nested expression structure | Sequential execution semantics (let, while, return) |
| TransitionTable | Petri net arcs | Function bodies | Type consumption and production relations only |

### Coexistence of the Lex₁ Path and Lex₂ Path

Conventional compilers follow a single path from high-level IR to low-level IR to target. laplan uses Lex₁ as the direct source for some languages (Haskell, OCaml, Gleam, Elixir) and lowers to Lex₂ for others (the remaining 17 languages + WASM). `has_functional_templates()` branches automatically based on whether a `functional {}` section exists in mapping.lex. See the "Lex₁ Path vs Lex₂ Path" section in [synthesis.md](synthesis.md).

## Key Types

### Declarations

| Type | Location | Role |
|---|---|---|
| `RuleBundle` | `rule.rs` | Collection of rule / const / assign / chain. Solver input. |
| `LexiconIr` | `lib.rs` | Normalized representation of lexicon declarations. `LexiconKind`, `LexiconObject`, `LexiconField`. |
| `Module` | `module.rs` | Aggregate for an entire `.lex` file. |
| `LibConfig` | `lib_config.rs` | cratis / face / member declarations. |
| `BuildConfig` | `build_config.rs` | Generation targets (`EmitTarget`, `BoundaryRule`). |
| `FamilyTable` | `family.rs` | family declarations (product, vectorize, etc.). |
| `Mapping` | `mapping.rs` | Language mapping (type_map, lowering, functional sections, etc.). |
| `RefinementDecl` | `refinement.rs` | Additional constraints on existing lexicons. |
| `VendorManifest` | `lib.rs` | vendored-json manifest. |

### Expressions and Statements

The IR represents functions in two layers. The functional expression written by the user as `rule.body` is received at Lex₁ and lowered to Lex₂ before being passed to code generation.

| Layer | Type | Use |
|---|---|---|
| Lex₁ | `FnExpr` (`fn_expr.rs`) | For functional languages (Haskell / OCaml / Gleam / Elixir) |
| Lex₂ | `Stmt` / `Expr` (`stmt_expr.rs`) | For imperative languages (remaining 17 languages + WASM) |

## Lex₁: FnExpr

Functional-style expression representation. Supports let-in, lambda, fold, filter, map-transform, case-of, Recurse (safe recursive), and more.

```rust
pub enum FnExpr {
    Var(String),
    StringLit(String),
    IntLit(i64),
    BoolLit(bool),
    App(String, Vec<FnExpr>),
    Lambda(Vec<(String, String)>, Box<FnExpr>),
    Fold { f, init, collection },
    Recurse { base_case, base_value, step, state },
    Filter(Box<FnExpr>, Box<FnExpr>),
    MapTransform(Box<FnExpr>, Box<FnExpr>),
    LetIn { bindings, body },
    Case { target, branches },
    FieldAccess(Box<FnExpr>, String),
    Construct(String, Vec<(String, FnExpr)>),
    ListLit(Vec<FnExpr>),
    MapFromList(Box<FnExpr>),
    MapLookup(Box<FnExpr>, Box<FnExpr>),
    MapMember(Box<FnExpr>, Box<FnExpr>),
    Null,
    IsNothing(Box<FnExpr>),
    FromMaybe(Box<FnExpr>, Box<FnExpr>),
    Not(Box<FnExpr>),
    BinaryOp(String, Box<FnExpr>, Box<FnExpr>),
    ErrorRaise(String),
    Tuple(Vec<FnExpr>),
    Fst(Box<FnExpr>),
    Snd(Box<FnExpr>),
    Head(Box<FnExpr>),
    NullCheck(Box<FnExpr>),
    Concat(Box<FnExpr>, Box<FnExpr>),
}
```

`Recurse` is the representation of safe recursive (`recursive.bounded` / `recursive.decreasing`). Lowering to While + Mut converts recursive patterns into loops.

### resolver.lex: KDL Notation for FnExpr

`axiom/resolver.lex` is a `.lex` variant that writes FnExpr directly in KDL. It declaratively defines the 9 runtime resolver functions (loadRecipes, dispatchRecipeStep, checkNeeds, classifyCandidate, failGoalUnreachable, resolveFromCandidates, executeRecipe, resolve, fetchGoal) and serves as the source of truth.

`parse_resolver_lex()` (`fn_expr.rs`) provides a semantic interpreter for KDL → `Vec<FnDef>`. The structure places a mapping layer to FnExpr variants on top of the KDL parser ([`neco-kdl`](https://github.com/barineco/neco-crates/tree/main/neco-kdl)) and uses only standard KDL syntax.

KDL node names correspond to key names in `functional {}` templates:

| KDL node | FnExpr variant | Child nodes |
|---|---|---|
| `fn "name" { ... }` | `FnDef` | `param` (direct children), `return-type`, body = remaining single expr |
| `var "local.x"` / `$local.x` | `Var` | - (`$` shorthand is syntactic sugar for `var`) |
| `string "..."` | `StringLit` | - |
| `int 42` | `IntLit` | - |
| `bool #true` | `BoolLit` | - |
| `null-literal` | `Null` | - |
| `app "f" { expr* }` | `App` | positional children = args |
| `lambda { param* expr }` | `Lambda` | `param` (direct children), body = sole non-param child |
| `fold { expr expr expr }` | `Fold` | 3 positional: fn, init, collection |
| `recurse { expr expr expr binding* }` | `Recurse` | 3 positional (base-case, base-value, step) + `binding` children = state |
| `filter { expr expr }` | `Filter` | 2 positional: predicate, collection |
| `map-transform { expr expr }` | `MapTransform` | 2 positional: transform, collection |
| `let-in { binding* expr }` | `LetIn` | `binding` children (value is direct child) + non-binding = body |
| `case { expr branch* }` | `Case` | non-branch child = target + `branch` children |
| `field "name" { expr }` | `FieldAccess` | sole child = target |
| `construct "C" { field "f" { expr } }` | `Construct` | `field` children (value is direct child) |
| `list { expr* }` | `ListLit` | positional children |
| `tuple { expr* }` | `Tuple` | positional children |
| `map-from-list { expr }` | `MapFromList` | sole child |
| `map-lookup { expr expr }` | `MapLookup` | 2 positional: target, key |
| `map-member { expr expr }` | `MapMember` | 2 positional: target, key |
| `is-nothing { expr }` | `IsNothing` | sole child |
| `from-maybe { expr expr }` | `FromMaybe` | 2 positional: default, value |
| `not { expr }` | `Not` | sole child |
| `binary op="+" { expr expr }` | `BinaryOp` | 2 positional: left, right |
| `error-raise "msg"` | `ErrorRaise` | - |
| `fst { expr }` | `Fst` | sole child |
| `snd { expr }` | `Snd` | sole child |
| `head { expr }` | `Head` | sole child |
| `null-check { expr }` | `NullCheck` | sole child |
| `concat { expr expr }` | `Concat` | 2 positional: left, right |

The `branch` in case specifies the pattern via properties: `constructor="Name"` (bound variables are children `bind "var"`), `tuple=#true`, `wildcard=#true`.

### Identifier NSID namespace

String identifiers in FnExpr KDL carry a prefix indicating their origin layer:

| Prefix | Meaning | Example |
|---|---|---|
| `local.` | Function parameter / let binding | `$local.goal` |
| `lang.` | Language primitive / runtime utility | `app "lang.mapGetRequired"` |
| `cli.` | CLI runtime primitive | `app "cli.call"`, `$cli.ctx` |
| (none) | Self-reference within the same file | `app "resolve"` |

At emit time, the prefix is stripped and only the base name appears in the generated language-specific code.

Parse results flow directly into the existing lowering (`fn_to_lex2`) and `template_engine_fn` (`emit_fn_expr`), generating resolver code for all 21 languages + 4 Lex₁ languages. `runtime_program_fn.rs` loads resolver.lex via `include_str!("../../../axiom/resolver.lex")` and returns it as `Vec<FnDef>` through `functional_resolve_program()`.

## Lex₂: Stmt / Expr

Imperative-style statements and expressions. Covers if / for / while / let / mut / return / continue, plus atomic operations and WASM-specific store operations.

```rust
pub enum Stmt {
    Let { pattern, ty, value },
    Mut { name, ty, value },
    Assign { target, value },
    If { cond, then_body, else_body },
    For { var, collection, body },
    While { cond, body },
    Return(Expr),
    Continue,
    StoreOp(String, Expr, Expr),
    AtomicStore { mnemonic, addr, value },
    Match { target: Expr, arms: Vec<(Pattern, Vec<Stmt>)> },
    Expr(Expr),
    Raw(String),
}

pub enum Expr {
    Var, StringLit, IntLit, BoolLit, I64Const,
    UnaryOp, BinaryOp, Tuple, Null,
    MapGet, MapContains, FieldAccess,
    ListEmpty, ListAppend, NewList, CopyMap,
    Call, Construct,
    IsNull, Not, First, ErrorRaise,
    AtomicLoad, AtomicRmwAdd, AtomicWait, // ...
    Match { target: Box<Expr>, arms: Vec<(Pattern, Expr)> },
    MethodChain { receiver: Box<Expr>, calls: Vec<(String, Vec<Expr>)> },
    IfLetTest { pattern: Box<Pattern>, rhs: Box<Expr> },
    IfElse { cond: Box<Expr>, then_body: Vec<Stmt>, else_body: Vec<Stmt> },
    Raw(String),
}
```

Roles of the additional variants:

| variant | Role |
|---|---|
| `Stmt::Let` | `let <pattern>: <ty> = <value>;`. The LHS holds a `Pattern`, letting a plain binding (`let x = v;`) and a destructuring let (`let StructName { f1, f2 } = v;`) share a single path. |
| `Stmt::Match` | `match <target> { <pattern> => <body>, ... }` statement form. Each arm is `(Pattern, Vec<Stmt>)`; a single-expression arm is wrapped in e.g. `Stmt::Return` so the arm body is always `Vec<Stmt>`. |
| `Stmt::Expr(Expr)` | Expression in statement position (method-call-stmt / try-stmt / macro-call-stmt). Emitted as `{expr};`. |
| `Stmt::Raw(String)` | Verbatim fallback for a statement that cannot be structured. Its presence in inverse artifacts surfaces unhandled sites. |
| `Expr::Match` | match in expression position. The arm body is a single `Expr`; a block form either uses its final expression or falls back to `Expr::Raw`. |
| `Expr::MethodChain` | Represents `receiver.call1(args1).call2(args2)...`. Target of iterator-idiom recognition. |
| `Expr::IfLetTest` | The condition part of `if let <pattern> = <rhs>`. Used as `Stmt::If { cond: Expr::IfLetTest { .. }, .. }`. |
| `Expr::IfElse` | if-else in expression position (e.g. `let x = if cond { 1 } else { 2 };`). Block bodies are kept as `Vec<Stmt>`; independent from `Stmt::If` which covers the statement-position form. |
| `Expr::Raw(String)` | Verbatim fallback for an expression that cannot be structured (`await`, `unsafe { ... }`, macro invocations, etc.). |

`RuntimeFn` is the Lex₂ function definition and holds `name`, `params`, `return_type`, and `body: Vec<Stmt>`.

### Pattern

Structured pattern for match arms and the LHS of `if let`. Held by Lex₂ match structures; independent from Case branches on the Lex₁ side.

```rust
pub enum Pattern {
    Wildcard,
    Binding { name: String, mutable: bool },
    VariantUnit(String),
    VariantTuple { variant: String, binds: Vec<Pattern> },
    VariantStruct { variant: String, fields: Vec<(String, Pattern)> },
    Struct { name: String, fields: Vec<(String, Pattern)> },
    Literal(String),
    Guarded { pattern: Box<Pattern>, guard: Box<Expr> },
    Raw(String),
}
```

| variant | Surface form |
|---|---|
| `Wildcard` | `_` |
| `Binding` | `x` / `mut x` |
| `VariantUnit` | Variant without binds (`None`, `Ok(_)`-like) |
| `VariantTuple` | `Some(x)` / `Ok(value)` / `Err(e)` |
| `VariantStruct` | sum-type variant destructuring inside match arms (e.g. the `Point { x, y }` part of `Some(Point { x, y })`) |
| `Struct` | Product-type struct destructuring itself (e.g. the LHS of `let StructName { f1, f2 } = v;`). Distinct in context from `VariantStruct`. |
| `Literal` | literals such as `0`, `"str"`, `true` |
| `Guarded` | match arm with guard: `<pattern> if <guard>` |
| `Raw(String)` | Verbatim fallback when the pattern cannot be structured |

## Lowering: Lex₁ → Lex₂

`fn_to_lex2(def: &FnDef) -> RuntimeFn` in `lowering.rs` is the entry point for lowering.

```mermaid
flowchart LR
    src[FnExpr] --> classify{kind}
    classify -- LetIn --> bindings[binding sequence → Vec&lt;Stmt&gt;]
    classify -- Fold --> fold[accumulator + For]
    classify -- Filter --> filter[conditional branch + ListAppend]
    classify -- MapTransform --> mapt[For + transform]
    classify -- Recurse --> recurse[While + Mut]
    classify -- Case --> case[If chain]
    classify -- other --> expr[Return Expr]
    bindings --> stmts[Vec&lt;Stmt&gt;]
    fold --> stmts
    filter --> stmts
    mapt --> stmts
    recurse --> stmts
    case --> stmts
    expr --> stmts
```

| Source FnExpr | Lowered to | Notes |
|---|---|---|
| `LetIn` | `Let` / `Mut` sequence | `Let` or `Mut` depending on binding.ty |
| `Fold` | `Let acc` + `For` | Accumulator made mutable |
| `Filter` | `NewList` + `For` + `If` + `ListAppend` | Conditional test per element |
| `MapTransform` | `NewList` + `For` + `ListAppend` | Transform applied per element |
| `Recurse` | `Mut state` + `While` | Loop until base_case is satisfied |
| `Case` | `If` chain | Branch on constructor / tuple patterns |
| Other | `Return` + `Expr` | Pass-through conversion |

The Lex₂ path is composed entirely of automatic lowering from Lex₁ IR.

## Module / Cratis IR Representation

`Module` in `module.rs` aggregates a single file. `LibConfig` (`lib_config.rs`) represents cratis and holds `members`, `faces`, `provides`, and `requires`.

```rust
pub struct CratisConfig {
    pub name: String,
    pub version: u32,
    pub members: Vec<CratisSource>,
    pub faces: Vec<FaceConfig>,
    // provides / requires / axiom
}
```

cratis acts as both a standalone package and a workspace (determined by the presence of members). See [guide/cratis.md](../guide/cratis.md) for details.

## Feature Gate

| Feature | Enabled functionality |
|---|---|
| `filesystem` (default) | Functions depending on `std::fs` (`paths`, `github_fetch`, `load_bundled_manifest`) + `atproto-lexicon-vendored` |

`--no-default-features` enables building for WASM targets. The parser core (`parse_base_json`, `parse_kdl_lexicons_native`, `parse_rule_kdl`, `elaborate`, `rule_bundle_to_canonical`) is always available.
