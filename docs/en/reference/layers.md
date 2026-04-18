# .lex Top Nodes and Layer Classification

## Sequence

```
type      type existence
lexicon   Lex₀ (lex)        type structure (input/output definitions)
morph     Lex₁ (solve)      morphisms between types (solver search target)
─── solver boundary ───
func      Lex₂ (constrain)  structural constraints (solver decision material)
pkg       Lex₃ (package)    package management
```

Each layer treats the layer below it as a trusted axiom. Prefixes are optional (for backwards compatibility).

The **solver boundary** between Lex₁ and Lex₂ is the key to convergence: the solver searches only Lex₁, and uses Lex₂ as the basis for pruning, identification, and derivation. Without this boundary, the search space explodes.

## type: Types

The definition of types themselves. The foundation for all layers.

| Notation | Example |
|---|---|
| `type` | `type i32` |

## lexicon (Lex₀ lex): Type Structure

Places in the Petri net. Defines the structure of types and their inputs and outputs.

| Notation | Example | Description |
|---|---|---|
| `lexicon` | `lexicon i32.add version=1 { ... }` | Input/output definition for a type |

## rule (Lex₁ solve): Morphisms

Transitions in the Petri net. **Searched by the solver.**

| Notation | Short form | Description |
|---|---|---|
| `rule` | `rule` | Requirements and outputs |
| `morph.const` | `const` | Constant binding. Initial marking (`() → T`). Not reassignable |
| `morph.assign` | `assign` | Variable binding. Token flow (`() → T`). Reassignable |
| `rule.inverse` | `inverse` | Inverse morphism (pruning) |
| `rule.derives` | `derives` | Derivation from an existing morphism |
| `morph.chain` | `handler`/`chain` | Manual path specification |
| `morph.refinement` | `refinement` | Adding constraints to an existing lexicon |

## constrain (Lex₂): Structural Constraints

Places structural constraints on top of Lex₁ morphisms. **Not searched by the solver.** Used as the basis for pruning, identification, and derivation rules.

| Notation | Short form | Description |
|---|---|---|
| `func` | `mapping` | Language conversion rules |
| `func.family` | `family` | Type family (a closed set of types sharing isomorphic operations; the source of vectorize/product derivation) |
| `func.law` | `law` | Algebraic laws |
| `func.dual` | `dual` | Symmetry of bidirectional queries |
| `func.invariant` | `invariant` | Invariants (count consistency, etc.) |

## package (Lex₃): Package Management

Bundles Lex₂ and below into packages. Manages connections with the outside.

| Notation | Short form | Description |
|---|---|---|
| `pkg.import` | `import` | Type boundary for external implementations |
| `pkg.cratis` | `cratis` | Package / workspace declaration |

`cratis` serves as both a standalone package and a workspace. Determined by the presence of `members`:

```kdl
// Standalone package
cratis "encrypted-dm" version=1 {
    provides { ... }
    requires { ... }
}

// Workspace (has members)
cratis "my-app" {
    members {
        "atproto-client" path="client/cratis.lex"
        "atproto-server" path="server/cratis.lex"
    }
    faces {
        face "client" { emit "atproto-client"; axiom "atproto-server" }
        face "server" { emit "atproto-server"; axiom "atproto-client" }
    }
}
```

## Decision Criteria

| Question | Layer |
|---|---|
| Is it a type existence declaration? | type |
| Is it an input/output structure definition? | Lex₀ lex (lexicon) |
| Does the solver search it as a transition? | Lex₁ solve (rule) |
| Does the solver not search it, but use it as decision material? | Lex₂ constrain |
| Is it package / external connection management? | Lex₃ package |

## Correspondence with the axiom/ Directory

```
axiom/
  i32/            Lex₀ + Lex₁ (type definitions + operations)
  str/            Lex₀ + Lex₁
  crypto/         Lex₀ + Lex₁
  category/       Lex₂ + Lex₃ (`cratis.lex` is the package root, compose/dual/lift are Lex₂)
  algebra/        Lex₀ + Lex₁ + Lex₂ (operations + family)
  resolver.lex    Lex₁ (FnExpr KDL. 7 runtime resolver functions)
  target/         Lex₂ + Lex₃ (`cratis.lex` is the workspace root; `lang/`, `binary/`, `bind/` each have their own `cratis.lex`)
    lang/           mapping.lex (type mapping table) + morph.lex + type.lex
    binary/
    bind/
```

`resolver.lex` is a .lex variant that directly describes FnExpr in KDL. While ordinary .lex handles declarations such as lexicon / rule / mapping, `resolver.lex` describes Lex₁ function definitions using `fn` nodes. `parse_resolver_lex()` converts KDL → `Vec<FnDef>`, and resolver code for all languages is generated via lowering. See the "resolver.lex" section in [architecture/ir.md](../architecture/ir.md) for details.

Each `axiom/*/cratis.lex` is a Lex₃ package root, and `axiom/target/cratis.lex` is the workspace root that bundles `lang`, `binary`, and `bind`.
The `provides` here is package metadata and is independent of endpoint loading.
A single directory may contain multiple layers. Layer separation is determined by node prefix, not by directory.
