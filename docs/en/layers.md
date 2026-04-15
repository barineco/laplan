# .lex Top-Level Nodes and Layer Classification

## Ordering

```
type    The existence of a type
lex     Lex₀  Type structure (input/output definitions)
rule    Lex₁  Rules between types (solver search targets)
─── solver boundary ───
func    Lex₂  Structural constraints (solver metadata)
pkg     Lex₃  Package management
```

Each layer treats the layers below it as trusted axioms. The prefix is optional (for backward compatibility).

The **solver boundary** between Lex₁ and Lex₂ is the key to convergence: the solver only explores Lex₁, using Lex₂ as pruning rules, equivalence classes, and derivation sources. Without this boundary, the search space explodes.

## type: Types

Definitions of types themselves. The foundation of all layers.

| Notation | Example |
|---|---|
| `type` | `type i32` |

## lex (Lex₀): Type Structure

Places in the Petri net. Defines the structure of types and their inputs/outputs.

| Notation | Example | Description |
|---|---|---|
| `lex` | `lex i32.add version=1 { ... }` | Input/output definition for a type |

## rule (Lex₁): Rules

Transitions in the Petri net. **Searched by the solver.**

| Notation | Short form | Description |
|---|---|---|
| `rule` | | Requirements and outputs |
| `rule.const` | `const` | Constant binding. Initial marking (`() → T`). Immutable |
| `rule.assign` | `assign` | Variable binding. Token flow (`() → T`). Mutable |
| `rule.inverse` | `inverse` | Inverse operation (pruning) |
| `rule.derives` | `derives` | Derivation from existing rules |
| `rule.chain` | `handler`/`chain` | Manually specified path |
| `rule.refinement` | `refinement` | Additional constraints on existing lex |

## func (Lex₂): Structural Constraints

Places structural constraints assuming the rules of Lex₁. **The solver does not search this layer.** Used as pruning rules, equivalence classes, and derivation sources.

| Notation | Short form | Description |
|---|---|---|
| `func` | | Language transformation rules |
| `func.family` | `family` | Type family (a closed set of types sharing isomorphic operations; source for vectorize/product derivations) |
| `func.law` | `law` | Algebraic laws |
| `func.dual` | `dual` | Bidirectional query symmetry |
| `func.invariant` | `invariant` | Invariants (count consistency, etc.) |

## pkg (Lex₃): Package Management

Bundles and packages everything at Lex₂ and below. Manages external connections.

| Notation | Short form | Description |
|---|---|---|
| `pkg.import` | `import` | Type boundary for external implementations |
| `pkg.cratis` | `cratis` | Package / workspace declaration |

`cratis` serves as both a single package and a workspace. Determined by the presence of `members`:

```kdl
// Single package
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
| Declaring the existence of a type? | type |
| Defining input/output structure? | Lex₀ (lex) |
| Does the solver search it as a transition? | Lex₁ (rule) |
| Not searched by solver, but used as metadata? | Lex₂ (func) |
| Package / external connection management? | Lex₃ (pkg) |

## Correspondence with the axiom/ Directory

```
axiom/
  i32/            Lex₀ + Lex₁ (type definitions + operations)
  str/            Lex₀ + Lex₁
  crypto/         Lex₀ + Lex₁
  category/       Lex₂ (compose, dual, lift)
  algebra/        Lex₀ + Lex₁ + Lex₂ (operations + family)
  target/         Lex₂ (language/binary/binding transformation rules)
    lang/           mapping.lex (type mapping) + rule.lex + type.lex
    binary/
    bind/
```

A single directory may contain multiple layers. Layer classification is determined by the node prefix, not the directory.
