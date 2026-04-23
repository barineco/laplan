# .lex Top Nodes and Layer Classification

## Sequence

```
type      type existence
lexicon   Lex₀ (lex)        type structure (input/output definitions)
rule      Lex₁ (solve)      morphisms between types (solver search target)
─── solver boundary ───
constrain Lex₂              structural constraints (solver decision material)
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
| `rule.const` | `const` | Constant binding. Initial marking (`() → T`). Not reassignable |
| `rule.assign` | `assign` | Variable binding. Token flow (`() → T`). Reassignable |
| `rule.inverse` | `inverse` | Inverse morphism (pruning) |
| `rule.derives` | `derives` | Derivation from an existing morphism |
| `rule.chain` | `handler`/`chain` | Manual path specification |
| `rule.refinement` | `refinement` | Adding constraints to an existing lexicon |

## constrain (Lex₂): Structural Constraints

Places structural constraints on top of Lex₁ morphisms. **Not searched by the solver.** Used as the basis for pruning, identification, and derivation rules.

| Notation | Short form | Description |
|---|---|---|
| `constrain.mapping` | `mapping` | Language conversion rules |
| `constrain.family` | `family` | Type family (a closed set of types sharing isomorphic operations; the source of vectorize/product derivation) |
| `constrain.law` | `law` | Algebraic laws |
| `constrain.dual` | `dual` | Symmetry of bidirectional queries |
| `constrain.invariant` | `invariant` | Invariants (count consistency, etc.) |

## package (Lex₃): Package Management

Bundles Lex₂ and below into packages. Manages connections with the outside.

| Notation | Short form | Description |
|---|---|---|
| `pkg.import` | `import` | Type boundary for external implementations |
| `pkg.cratis` | `cratis` | Package / workspace declaration |

## meta (Lex₃): Tool Metadata

A Lex₃ top node parallel to `pkg`. Stores operational metadata required by laplan tooling, completely separated from the body `.lex` (logic) by namespace and physical file.

Supported namespaces:

| nsid prefix | Purpose |
|---|---|
| `view.graph.<nsid>` | Node coordinates and viewport state for the Petri net graph UI |

`meta view.graph.<nsid>` saves node positions and viewport state for a body `.lex`. The `refs.target` field holds an explicit link to the body nsid for rename resilience.

```kdl
meta "view.graph.car.parse_v1" {
  refs {
    target "car.parse_v1"
  }
  nodes {
    node "input.data" x=100 y=50
  }
  viewport center-x=200 center-y=50 view-size=400
}
```

Files are placed under `view/graph/<nsid>.lex`, separated from the body `.lex` at the directory level. Parsing is handled by `parse_meta_kdl()` in `compiler/ir`. See [reference/axiom-view.md](axiom/view.md) for type declarations.

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
| Is it tool operational metadata (coordinates, viewport, etc.)? | Lex₃ meta |

## Correspondence with the axiom/ Directory

```
axiom/
  i32/            Lex₀ + Lex₁ (type definitions + operations)
  str/            Lex₀ + Lex₁
  crypto/         Lex₀ + Lex₁
  category/       Lex₂ + Lex₃ (`cratis.lex` is the package root, compose/dual/lift are Lex₂)
  algebra/        Lex₀ + Lex₁ + Lex₂ (operations + family)
  resolver.lex    Lex₁ (FnExpr KDL. 9 runtime resolver functions)
  target/         Lex₂ + Lex₃ (`cratis.lex` is the workspace root; `lang/`, `binary/`, `bind/` each have their own `cratis.lex`)
    lang/           mapping.lex (type mapping table) + morph.lex + type.lex
    binary/
    bind/
  view/           Lex₀ (cratis "view". Position / Size / graph sub-cratis)
    graph/          Lex₀ (NodeBox / EdgeRoute / ViewportTransform)
```

`resolver.lex` is a .lex variant that directly describes FnExpr in KDL. While ordinary .lex handles declarations such as lexicon / rule, `resolver.lex` describes Lex₁ function definitions using `fn` nodes. `parse_resolver_lex()` converts KDL → `Vec<FnDef>`, and resolver code for all languages is generated via lowering. See the "resolver.lex" section in [architecture/ir.md](../architecture/ir.md) for details.

Each `axiom/*/cratis.lex` is a Lex₃ package root, and `axiom/target/cratis.lex` is the workspace root that bundles `lang`, `binary`, and `bind`.
The `provides` here is package metadata and is independent of endpoint loading.
A single directory may contain multiple layers. Layer separation is determined by node prefix, not by directory.
