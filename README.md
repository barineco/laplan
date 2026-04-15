# Laplan: Lexicon as Programming Language

[日本語](README-ja.md)

> **Note:** This document includes planned features and designs. Details may change at release.

A metatype programming language that declares types and relations in KDL format and synthesizes them via Petri net path resolution.

## Metatype Programming Language

A Lexicon declares API endpoint inputs and outputs as types.

Laplan turns the act of "defining types" itself into a language: a language of types of types (metatypes).

Given types `did` and `profile`, the question "is there a path from did to profile?" becomes valid, and the solver searches for paths automatically. Types alone leave paths under-constrained; implicit preconditions like JWT tokens, expiration, and resource ownership live beyond API definitions. Declaring this tacit knowledge as rule constraints lets the solver discover only valid paths.

This path search is formalized as a Petri net reachability problem. The solver's design draws from Lean's tactic system: the more abstract the constraints (law, rule), the more powerfully the solver works; the more concrete the procedure, the more the programmer specifies manually. This mirrors the relationship between `ring` / `simp` and manual `apply`.

That paths serve as reachability proofs, that constraints change reachability, and that annotations are non-recoverable from types alone have been formally verified in Lean 4 / Mathlib ([lean-lexicon](https://github.com/barineco/lean-lexicon)). Related: [lean-eigenraum](https://github.com/barineco/lean-eigenraum) (formal verification of vibrational energy transport, 136 theorems, zero sorry).

## What This Enables

- Type safety across client/server boundaries
  Declare auth token validation as type constraints; catch errors before making an API call
- Explicit preconditions
  Consolidate tacit knowledge (JWT in HTTP headers, token expiry) scattered across code into declarations
- Framework-independent
  Unified type declarations free from framework-specific types (axum, Express, etc.)
- Automatic SIMD / GPU / threading optimization
  Optimal instruction sequences discovered as a byproduct of path search
- 22-language conversion + inverse
  Connect to existing workflows; borrowing only a subset is a supported use case
- Exhaustive path enumeration and diagnostics
  Finite search space enables listing all paths and reporting missing information

---

## Three Formats

Comparing the same API definition (`app.bsky.actor.getProfile`) in three formats:

### `.json` (Lexicon JSON: AT Proto canonical source)

```json
{
  "lexicon": 1,
  "id": "app.bsky.actor.getProfile",
  "defs": {
    "main": {
      "type": "query",
      "parameters": {
        "type": "params",
        "required": ["actor"],
        "properties": {
          "actor": {
            "type": "string",
            "format": "at-identifier"
          }
        }
      },
      "output": {
        "encoding": "application/json",
        "schema": { "type": "ref", "ref": "app.bsky.actor.defs#profileViewDetailed" }
      }
    }
  }
}
```

### `.kdl` (Lexicon KDL: faithful human-readable projection of JSON)

```kdl
lexicon app.bsky.actor.getProfile version=1 {
    query {
        params {
            (string)actor required=#true format=at-identifier
        }
        output "app.bsky.actor.defs#profileViewDetailed" encoding="application/json"
    }
}
```

### `.lex` (Laplan KDL: computational language)

```kdl
lex app.bsky.actor.getProfile version=1 xrpc=query {
    params { (str)actor format=at-identifier }
    output "app.bsky.actor.defs#profileViewDetailed" encoding="application/json"
}

rule "app.bsky.actor.getProfile" {
    requires input="actor"
    produces output="profile"
}
```

In Laplan, `required` is the default; `?` marks optional. The `query` / `procedure` distinction is preserved via `xrpc=` annotation. Type names use Laplan canonical forms (`str`, `i32`, `f64`, etc.). Declaring a rule lets the solver resolve paths.

The presence or absence of `xrpc=` exclusively separates two worlds:

| | `xrpc=query\|procedure\|subscription` | No `xrpc` |
|---|---|---|
| Namespace | NSID (reversed domain) | Laplan namespace (free-form) |
| Required default | optional (no `?` needed, Lexicon-compatible) | **required** (`?` for optional) |
| Round-trip | `.json <-> .kdl <-> .lex` fully reversible | No counterpart in Lexicon JSON |
| Example | `app.bsky.actor.getProfile` | `i32.add`, `io.print` |

---

### What Only Laplan Can Express

Lexicon JSON is one file = one NSID, with each endpoint isolated. KDL format (`.kdl` / `.lex`) allows **multiple lex declarations in a single file**, enabling related declarations to be managed together. The following cannot be described in Lexicon and become expressible for the first time in Laplan:

| Capability | Description | Keyword |
|---|---|---|
| Non-API functions | Arithmetic, logic, etc. not tied to HTTP | `in` / `out` |
| Rule declarations | What is needed and what is produced | `rule` |
| Derivation | Batch, compose, or transform existing declarations | `derives` |
| Composition pipelines | Multi-step handler chains | `handler` / `chain` |
| Inverse relations | Action/undo pairs | `inverse` |
| Dual queries | Forward/reverse commutativity conditions | `dual` |
| Algebraic laws | Group axioms, idempotency, homorules | `law` |
| Resource consumption | Exclusive token consumption (general Petri net) | `consumes` |

```kdl
// Arithmetic functions (not an API)
i32.add {
    in { (i32)a; (i32)b }
    out { (i32)result }
}

// Rule declarations (what is needed and what is produced)
rule "app.bsky.actor.getProfile" {
    requires input="actor"
    produces output="profile"
}

// Derivation from existing declarations (batch getProfile -> getProfiles)
derives "app.bsky.actor.getProfiles" from "app.bsky.actor.getProfile" via batch {
    singular-input "actor"   plural-input "actors"
    singular-output "profile" plural-output "profiles"
}

// Function composition pipelines
handler "ink.illo.dm.send" {
    chain {
        step "server.identity.resolve_pubkey" input="recipient-did" output="recipient-pubkey"
        step "crypto.vault.create_sealed_dm" input="content,recipient-pubkey" output="sealed"
        step "server.blockstore.write_record" input="sealed" output="uri,cid"
    }
}

// Inverse relations
inverse mute_actor {
    action "app.bsky.graph.muteActor"
    inverse "app.bsky.graph.unmuteActor"
    kind reversible
}

// Dual queries (commutativity conditions for forward/reverse)
dual follow_dual {
    record "app.bsky.graph.follow"
    forward "app.bsky.graph.getFollows" direction=outgoing key="subject"
    reverse "app.bsky.graph.getFollowers" direction=incoming key="subject"
}
```

Lexicon defines "what does this API accept and return." Laplan defines "how does this API relate to others, how can it be composed, and what does it presuppose."

---

## Overview

Laplan is a language inspired by the AT Protocol's [Lexicon](https://atproto.com/specs/lexicon) schema format. Just as Lexicon declares API endpoints, Laplan declares **typed interfaces for computation**.

Each declaration becomes a transition on a Petri net, and the solver automatically discovers paths that satisfy `requires` (needed tokens) and `produces` (generated tokens). The programmer writes "what I have and what I want," and Laplan resolves the path.

### Relationship to Lexicon

| | Lexicon | Laplan |
|---|---|---|
| Format | JSON (`.json`) / KDL (`.kdl`) | KDL (`.lex`) |
| Purpose | API schema definition | Typed description & composition of computation |
| Namespace | NSID (reversed domain) | Free (dot-separated) |
| Composition | None (call only) | dual / derives / Petri net |
| Required | optional by default | **required by default** |
| Type names | `string`, `integer`, `number` | `str`, `i32`, `i64`, `f64` |

The three formats are interconvertible:

| Conversion | Reversibility |
|---|---|
| `.json` <-> `.kdl` <-> `.lex` (with `xrpc=`) | Fully reversible (type names are bidirectionally converted) |
| `.lex` (without `xrpc=`) | Laplan native; no counterpart in Lexicon |

---

## Writing Declarations

### 1. Types (Place): The Foundation of Everything

The starting point of Laplan is type definitions. These correspond to **places** in the Petri net.

| Laplan | Description | Lexicon | WASM |
|---|---|---|---|
| `i32` | Signed 32-bit integer | `integer` | i32 |
| `i64` | Signed 64-bit integer | (none) | i64 |
| `f32` | 32-bit floating point | (none) | f32 |
| `f64` | 64-bit floating point | `number` | f64 |
| `bool` | Boolean (0/1) | `boolean` | i32 |
| `str` | UTF-8 string | `string` | host string |
| `bytes` | Byte sequence | `bytes` | host bytes |
| `any` | Arbitrary structured value | `unknown` | host value |

Structured types:

```kdl
(object)profileView {
    (str)did format=did                // type in parentheses + field name
    (str)handle format=handle
    (str)displayName? max-length=640   // ? marks optional
    (data.map)metadata?                // path in parentheses for custom types
}
```

### 2. Rule (Transition Declaration): Relations Between Types

Declare **reachability** between types. These are transitions in the Petri net. A rule declares only "what is needed and what is produced"; implementation is delegated to bridges.

```kdl
// Login: produces auth tokens
rule "com.atproto.server.createSession" {
    produces capability="access_jwt"
    produces capability="refresh_jwt" ttl=86400
}

// Token refresh: consumes refresh, produces new access
rule "com.atproto.server.refreshSession" {
    requires capability="refresh_jwt"
    consumes capability="refresh_jwt"      // consumed on use
    produces capability="access_jwt" ttl=3600
}

// Get profile: requires access_jwt
rule "app.bsky.actor.getProfile" {
    requires capability="access_jwt"
}
```

Token existence, expiry (`ttl`), and exclusive consumption (`consumes`) become type-level constraints. The solver automatically discovers the path `createSession → getProfile`. When `access_jwt` has expired, the solver works backwards from "what is missing" and inserts the re-authentication path (`refreshSession`) automatically. No hardcoded if-statements for token validation; the necessary steps are derived from declared constraints.

### 3. Functions (Transition Constraints): Concretizing Type Signatures

Rules alone only establish type relations. Functions give those paths **concrete type signatures**.

```kdl
app.bsky.actor.getProfile xrpc=query {
    in { (str)actor format=at-identifier }
    out { (ref)profile type="app.bsky.actor.defs#profileViewDetailed" }
}

data.json.parse {
    in { (str)input }
    out { (any)value }
    errors {
        error InvalidJson description="invalid JSON"
    }
}
```

Functions constrain transitions to "this goes in, this comes out." When constraints are sufficient, the solver's paths naturally converge to a single one.

### 4. Chain: Manual Path Pinning

Explicitly specify step ordering rather than relying on the solver's automatic paths.

```kdl
handler "ink.illo.dm.send" {
    chain {
        step "server.identity.resolve_pubkey" input="recipient-did" output="recipient-pubkey"
        step "crypto.vault.create_sealed_dm" input="content,recipient-pubkey" output="sealed"
        step "server.blockstore.write_record" input="sealed" output="uri,cid"
    }
}
```

### Types -> Rule -> Function -> Chain: Layering Constraints

| Layer | Declares | Style |
|---|---|---|
| **Type** (place) | What exists | Most declarative |
| **Rule** | What can reach what (direction) | |
| **Function** | What goes in, what comes out (type signature) | |
| **Chain** | What order to traverse (procedure) | Most imperative |

> Higher layers delegate to the solver; lower layers are specified by the programmer.

**If the upper layers suffice, the lower ones can be omitted.** If the solver finds a unique path from rules alone, neither functions nor chains are needed. If not, add constraints at lower layers.

At high levels (APIs), delegating to the solver via rules alone is natural. "I have a DID, I want a profile" is sufficient. Even at low levels (arithmetic), declaring algebraic laws via `law` lets the solver identify equivalent paths, and execution order and parallelism are derived from type dependencies. Chains are not theoretically required, but useful when expressing the goal as types is difficult, or when stating a procedure directly is clearer than writing structural constraints.

Frequently used solver paths can be pinned as chains. This eliminates solver path-discovery cost, yielding a deterministic function.

### Solver Behavior

The solver resolves Petri net reachability via **breadth-first search (BFS)**.

#### Three Concepts

| Concept | Definition | Example |
|---|---|---|
| **Marking** | Set of tokens currently held (on Petri net places) | `{ Capability("access_jwt"), Output("did"), SelfKey("self.repo") }` |
| **Goal** | Tokens to reach | `[ Output("profile") ]` |
| **Transition** | Checks requires, consumes consumed tokens, generates produces | see below |

| Transition | requires | produces |
|---|---|---|
| `resolve_handle` | _(none)_ | `Output("did")` |
| `getProfile` | `Output("did")` | `Output("profile")` |
| `getFollowers` | `Output("did")` | `Output("followers")` |

#### BFS Flow

> **Initial marking**: `{ Input("handle") }` | **Goal**: `[ Output("profile") ]`

0. **Depth 0** ... marking: `{ Input("handle") }`
   - Fireable: `resolve_handle` (requires=[] → always fireable)
1. **Depth 1** ... marking: `{ Input("handle"), Output("did") }`
   - Fireable: `getProfile` (requires=[did] ok)
   - Fireable: `getFollowers` (requires=[did] ok) ... both fireable, but...
2. **Depth 2**
   - via `getProfile` → `{ ..., Output("profile") }` ... **goal reached!**
   - via `getFollowers` → `{ ..., Output("followers") }` ... goal not reached

> **Result**: 1 path `[ resolve_handle → getProfile ]`

The solver tries all fireable transitions breadth-first and returns the shortest path to the goal. Only transitions whose produces match the goal remain in the path.

#### Combining Multiple cratis

When multiple cratis are loaded simultaneously, all transitions merge into a single TransitionTable. Where API surfaces between cratis touch, the solver discovers cross-cratis paths.

| cratis | Transition | requires | produces |
|---|---|---|---|
| `atproto/` | `resolve_handle` | _(none)_ | `did` |
| `bsky/` | `getProfile` | `did` | `profile` |
| `bsky-cli/` | `display_profile` | `profile` | _(display)_ |

> **Solver result**: `resolve_handle` → `getProfile` → `display_profile` (cross-cratis path)

Each cratis declares only its own provides / requires. The solver handles composition.

#### Path Convergence and Branching

**If produces differ, paths naturally converge to one.** In the example above, `getProfile` produces `profile` and `getFollowers` produces `followers`. If the goal is `profile`, only `getProfile` can be chosen.

Branching occurs **when multiple transitions produce the same goal**:

| Transition | requires | produces |
|---|---|---|
| `getProfileByDid` | `did` | `profile` |
| `getProfileByHandle` | `handle` | `profile` |

> **marking**: `{ did, handle }` | **goal**: `[ profile ]` → solver: **2 paths** (both produce profile)

This is a legitimate branch: "profile can be obtained from either a DID or a handle." Resolution methods:

1. **Narrow the marking**: With only `{ did }`, there's a single path via `getProfileByDid`
2. **Add requires**: Differentiate by attaching a capability to one
3. **Pin with chain**: Explicitly specify the desired path

Real-world case where 3 paths emerged: When solving Lexicon JSON as-is, the JWT token in the HTTP header was not part of the API definition, so the requires for `getProfile` / `getBlocks` / `getFollows` were all just `[did]`. Adding capability to the rules converged to 1 path.

**Multiple paths are design feedback.** When the solver returns branches, it's a signal to review the requires / produces declarations.

#### Unreachability and Missing Information Diagnosis

When the goal is unreachable, the solver classifies and reports what's missing:

| Diagnosis | Meaning | Example |
|---|---|---|
| **AlreadySatisfied** | Goal is already in the marking | |
| **Reachable** | Path found | `[resolve_handle -> getProfile]` |
| **Recoverable** | Reachable if some facts are supplied | "Reachable if did is available" |
| **NeedsUserAction** | User input required | "Please enter the actor's DID" |
| **PrunedByBoundary** | Blocked by boundary conditions | Auth / permissions / insufficient data |

`BoundaryKind` classifies blocking reasons into 3 types:
- **Capability**: No auth token (e.g., `access_jwt`)
- **Ownership**: Not your resource (e.g., attempting to write to another's repo)
- **Output**: Required data unavailable (e.g., DID unresolved)

This diagnosis presents "what you have, what's missing, and what would make it reachable" to the programmer.

#### Two-Layer Solver

The solver operates in two layers. The same BFS algorithm is reused with different Fact types.

| Layer | Target | Input | Output | Fact examples |
|---|---|---|---|---|
| Layer 1: rule solver | API paths | marking (capabilities etc.) + goal | Recipe (NSID sequence) | `Capability("access_jwt")`, `Output("profile")` |
| Layer 2: instruction solver | WASM instruction selection | value-level marking + goal | Recipe (opcode sequence) | `Value { ty: "f64", name: "a" }`, `SimdValue { ty: "f64x2", name: "ab" }` |

Layer 2 places SIMD transitions (generated from family's `vectorize` derives) alongside scalar transitions in the same table. When BFS discovers the shortest path, SIMD paths are automatically selected if shorter than scalar paths.

```
// Example: sum_of_squares
// Scalar path  (depth 3): f64.mul → f64.mul → f64.add
// SIMD path    (depth 2): f64x2.mul → f64.add
// → BFS discovers the depth-2 SIMD path first
```

**SIMD optimization is not an optimization pass, but a consequence of shortest-path discovery on the Petri net.**

### 5. Family: Type Family for Isomorphic Operations

Declares isomorphic relations between types. A family is a closed set of types sharing the same operation signatures.

```kdl
family "Numeric" {
    members {
        "i32"   wasm="i32"
        "i64"   wasm="i64"
        "f32"   wasm="f32"
        "f64"   wasm="f64"
    }
    signature "add" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "sub" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "mul" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "neg" { in { (Self)a }          out { (Self)result } }
}
```

A family declares a closed set of types sharing isomorphic operations. The `vectorize` derives pattern automatically derives SIMD transitions from family signatures:

```
Numeric family:
  integer  --- add --> integer       vectorize (functor)
  number   --- add --> number    →   f64x2 --- add --> f64x2
  float32  --- add --> float32       f32x4 --- add --> f32x4
```

`vectorize` is a functor that performs "lifting from the scalar category to the SIMD category." The instruction solver's BFS naturally discovers these lifted transitions as "shorter paths."

### Derives: Derivation from Existing Declarations

| Strategy | Description | Example |
|---|---|---|
| `compose` | Combine existing functions into a new one | `sum_of_squares` from `multiply` + `add` |
| `binary_accumulate` | Derive iterative from binary operations | `mod_mul` from `mod_add` |
| `conditional` | Conditional branching | `abs` via sign check |
| `host_import` | Import from host environment | `fp_add` from WASM host |
| `vectorize` | Auto-derive SIMD transitions from family isorules | `f64x2.add`, `f32x4.mul` from Numeric family |

```kdl
// Composition: define new functions by combining existing ones
derives "i32.sum_of_squares" via compose {
    sources "i32.mul" "i32.add"
    steps {
        step "i32.mul" input="a,a" output="sq_a"
        step "i32.mul" input="b,b" output="sq_b"
        step "i32.add" input="sq_a,sq_b" output="result"
    }
}

// Accumulation: derive iterative operations from binary operations (e.g., addition -> multiplication)
derives "algebra.modular.mod_mul" from "algebra.modular.mod_add" via binary_accumulate {
    identity 0
    halve "i64.shr1"
    test "i64.test_lsb"
}

// Conditional branching
derives "i32.abs" via conditional {
    cond "i32.lt_s(a, 0)"
    then "i32.neg(a)"
    else "a"
}

// Host import
derives "algebra.field.gf256.fp_add" via host_import {
    module "algebra.field.gf256" name "algebra.field.gf256.fp_add"
}

// vectorize: auto-derive SIMD transitions from family isorules
derives "simd.f64x2" from family="Numeric" member="number" via vectorize {
    lane-width 2
    target "wasm"
}
derives "simd.f32x4" from family="Numeric" member="float32" via vectorize {
    lane-width 4
    target "wasm"
}
```

The `target` parameter of `vectorize` specifies the parallelization target:

| target | Meaning | lane-width interpretation |
|---|---|---|
| `"wasm"` | WASM SIMD v128 instructions | Lane count (2 or 4) |
| `"wgpu"` | GPU compute shader dispatch | workgroup_size |
| `"thread"` | CPU thread pool | Thread count |

### Import: Type Boundaries for External Bridges

```kdl
import "neco-vault" {
    procedure "create_sealed_dm" {
        in {
            (str)content
            (bytes)recipient-pubkey
        }
        out {
            (bytes)ciphertext
            (bytes)ephemeral-pubkey
        }
    }
}
```

A bridge is a connection point to external implementations (e.g., Rust crates) at Laplan's type boundary. Only input/output types are declared; implementation is delegated externally.

---

## axiom: Standard Library

Built-in base declarations for Laplan. All with zero external dependencies.

> **The term axiom:** An `axiom` refers to declarations that belong to a layer **beyond** the category of `.lex` implementations and are trusted unconditionally. This includes standard cratisry primitives, values at the API boundary (HTTP headers, runtime-provided values), and functor specifications via `import` (external Rust crates, etc.). The correctness of axioms is not something Laplan's solver verifies; it is accepted as a premise. This mirrors the role of axioms in mathematics.

| Module | Contents |
|---|---|
| `i32/` | Arithmetic, bit operations |
| `i64/` | 64-bit arithmetic, bit operations |
| `f32/` | 32-bit floating point operations |
| `f64/` | 64-bit floating point operations |
| `bool/` | Logic operations |
| `str/` | String operations, patch, diff, wrap, fuzzy, path, derives |
| `bytes/` | Byte sequence operations |
| `array/`, `map/`, `tree/` | Collection operations |
| `json/`, `cbor/`, `kdl/` | Serialization formats |
| `cid/`, `car/` | Content-addressable data |
| `encoding/` | base64, base58 |
| `algebra/` | Modular arithmetic, Galois fields, matrices, family (Numeric), monoid, group |
| `category/` | compose, dual, lift, product, fold, restrict, law |
| `convert/` | Type conversion (i32_to_i64, etc.) |
| `random/` | Pseudo-random number generators |
| `crypto/` | hash (sha256/sha1/hmac), secp256k1, vault |
| `memory/` | Linear memory (WASM load/store), access |
| `effect/` | import |
| `integrity/` | assert |
| `io/` | print, time, rand |
| `time/`, `length/`, `mass/`, `temperature/`, `frequency/` | Dimensional quantities (duration, distance, etc.) |

---

## cratis: Domain-Specific Libraries

Independent `.lex` file collections, like crates. Each declares provides / requires in `cratis.lex`.

| cratis | Domain |
|---|---|
| `atproto/` | AT Proto (`com.atproto.*`) |
| `bsky/` | Bluesky (`app.bsky.*`) |
| `encrypted-dm/` | Encrypted DM |
| `bsky-cli/` | CLI command definitions |
| `pds-frontpage/` | PDS front page |

### cratis Structure

Example: `cratis/encrypted-dm/`

| Path | Role |
|---|---|
| `lex/cratis.lex` | provides / requires declarations |
| `lex/dm.lex` | Record types & procedure definitions |
| `lex/chain.lex` | Handler pipelines |
| `lex/rule.lex` | Rule declarations |
| `lex/import.lex` | Type boundaries for external bridges |
| `src/synth/` | synthesis output (e.g., Rust) |

`cratis.lex`:
```kdl
cratis "encrypted-dm" version=1 {
    description "Encrypted DM"
    namespace "ink.illo.dm"

    provides {
        procedure "ink.illo.dm.send"
        procedure "ink.illo.dm.list"
        record "ink.illo.dm.sealed"
    }

    requires {
        capability "access_jwt"
        import "neco-vault" features="nip17"
    }
}
```

---

## Composition

`cratis` serves as both a single package and a workspace. Determined by the presence of `members`:

> **cratis**: Latin for framework/lattice. The etymological root of English "crate". A frame holding lexicons.

| Form | Cargo | Determined by |
|---|---|---|
| Single cratis | `Cargo.toml` (single crate) | No `members` |
| Workspace cratis | `Cargo.toml [workspace]` | Has `members` |

Each cratis declares only its own provides/requires. The solver automatically connects interfaces and discovers cross-cratis paths.

A workspace cratis bundles multiple cratis and defines faces. A face determines "from this perspective, what to generate and what to trust as a premise."

```kdl
cratis "my-app" {
    members {
        "atproto-client" path="client/cratis.lex"
        "atproto-server" path="server/cratis.lex"
    }
    faces {
        face "client" {
            emit "atproto-client"
            axiom "atproto-server"
        }
        face "server" {
            emit "atproto-server"
            axiom "atproto-client"
        }
    }
}
```

- **emit**: synthesis target. Generate code for this cratis
- **axiom**: trusted premise. Reference the interface only; no code is generated

In the client face, the server side is an axiom (trusted to exist). In the server face, the client side is an axiom. The solver's search space changes depending on the face, even within the same workspace.

---

## Build

### Compiler and Codegen Built-in Resources

Language target transformation rules are placed as **built-in resources** of the compiler / synthesis under the `target.*` namespace. They are not placed in user projects.

| Category | Namespace | Description |
|---|---|---|
| Compiler | `target.binary.wasm` | WASM instruction mapping |
| Codegen | `target.lang.rust` | Rust type mapping, syntax patterns, code generation templates |
| Codegen | `target.lang.python` | Python (same) |
| Codegen | `target.lang.typescript` | TypeScript (same) |
| Codegen | ... | 21 languages total |

Each target consists of three files:

| File | Role |
|---|---|
| `mapping.lex` | Type mapping |
| `morph.lex` | Syntax patterns |
| `type.lex` | Type conversion |

These are rules for synthesis to know "how to write in this language," analogous to LLVM backend definitions inside `rustc`.

### WASM Binary Build

Generates WASM modules from axiom declarations + distributed derives/pattern definitions.

> `axiom/*.lex` → **compiler** → `.wasm`

| Compiler Stage | Input | Output |
|---|---|---|
| parser | `.lex` | AST |
| ir | AST | Intermediate representation |
| compile | IR | WASM (using `target.binary.wasm`) |

Derives expansion rules:

- `derives via compose` / `derives via binary_accumulate` → expanded into WASM instruction sequences
- `derives via host_import` → converted to WASM import sections
- `derives via vectorize` → auto-generates SIMD transitions from family signature x member x lane-width

### Compiler Options

| Flag | Effect | Solver layer |
|---|---|---|
| `--bake` | Enumerate all reachable paths from module provides/requires, auto-generate WASM functions as frozen chains. Zero user-authored chains, zero runtime solve cost | Layer 1 only |
| `--simd` | Pass `--bake` frozen chains through the instruction solver; auto-replace with SIMD when shorter than scalar | Layer 1 + Layer 2 |
| `--constant-time` | Exclude transitions with `branching` attribute from the solver. Only branchless paths are discovered | Transition filter |
| `--parallel` | Auto-detect independent transitions, build parallel execution DAG. Convert to WASM threads / wgpu dispatch / thread pool based on target | Petri net analysis |

The three flags are orthogonal and freely combinable:

```bash
# From places alone, generate SIMD-optimized + side-channel-safe WASM functions
laplan compile cratis/encrypted-dm/ --bake --simd --constant-time
```

`--bake` turns the solver into a linker. All symbol resolution is completed at compile time.
`--simd` selects SIMD as BFS shortest-path discovery. Not a cost function addition.
`--constant-time` is a transition table filter. The solver algorithm is shared.

### Multi-Language Codegen

From axiom + cratis declarations, synthesis references built-in target definitions to generate source code for each language.

> `axiom/*.lex` + `cratis/*.lex` → **synthesis** → `src/synth/*`

| Target | Output |
|---|---|
| `target.lang.rust` | `src/synth/*.rs` |
| `target.lang.python` | `src/synth/*.py` |
| ... | (same pattern for all languages) |

Supported languages (22 targets; 18 active, 3 deferred, 1 removed):

Maturity levels: L1: syntax lint pass, L2: type compilation pass, L3: package build pass, L4: runtime smoke pass.

| Language | Category | Maturity |
|---|---|---|
| Rust | Systems | L4+ (workspace: 104+ files, WASM output, inverse complete) |
| Python | Scripting | L4 |
| Go | Systems | L4 |
| Ruby | Scripting | L4 |
| Kotlin | JVM / Mobile | L4 |
| Java | JVM | L4 |
| Lua | Scripting | L4 |
| TypeScript | Web | L3 |
| Swift | Mobile | L3 |
| C# | .NET | L3 |
| Dart | Mobile | L3 |
| Zig | Systems | L3 |
| Clojure | JVM | L3 |
| Haskell | Functional | L2 |
| C++ | Systems | L2 |
| D | Systems | L2 |
| JavaScript | Web | L1 |
| PHP | Scripting | L1 |
| OCaml | Functional | deferred |
| Elixir | Functional | deferred |
| Gleam | Functional | deferred |

### cratis Output Location

Output location varies by synthesis target. Example for Rust target (`cratis/encrypted-dm/`):

| Path | Role | Source |
|---|---|---|
| `lex/*.lex` | Declarations | Input (source) |
| `src/synth/mod.rs` | Module root | synthesis output |
| `src/synth/handler.rs` | Handler implementations | synthesis output |
| `src/bridge.rs` | Bridge implementation | Source |
| `src/lib.rs` | Binds `synth/` as a crate | Source |
| `Cargo.toml` | Crate manifest | Source |

For Rust, `src/synth/` + `src/lib.rs` function as a crate. For WASM binaries, the output goes to `target/` (same as Cargo).

---

## Recommended Workflow

1. **Write Places** ... Type definitions (can also convert from Lexicon JSON)
2. **Pass to Solver** ... Specify "what I have and what I want"
3. **Observe convergence**
   - 1 path → done
   - Multiple paths → insufficient constraints
   - Unreachable → check missing facts
4. **Add what's missing** ... rules, capabilities, a few axiom lines
5. **Return to 1**

There is a proven track record of loading AT Protocol Lexicons (~100 NSIDs) as-is, adding just a few lines of capability for request headers, and having the solver automatically resolve all paths. The Lexicon designers weren't thinking about Petri nets, but because they honestly wrote types and inputs/outputs as API definitions, that alone was enough for paths to converge.

**Write places honestly, and paths emerge on their own.**

---

## Design Principles

1. **Types are the foundation.** If types exist, a Petri net emerges and path questions become valid. Functions are constraints on paths, not starting points.
2. **Declarations are the source of truth.** Code is generated from declarations. The existence of user-authored code implies insufficient declarations.
3. **Individuals are finite; the whole is Turing-complete.** The solver's search space is finite and always terminates, enabling enumeration and diagnosis of all paths. Each bridge (Rust crate, WASM, HTTP API) also has a finite interface. Yet the possibility space of their combinations reaches Turing completeness. Because each individual is explorable, the whole becomes soundly Turing-complete.
4. **Zero external dependencies.** axiom's bridge backing (neco-crates) is all pure Rust. Environment-agnostic.
5. **Interconvertibility.** `.json <-> .kdl <-> .lex` is fully reversible with `xrpc=`. Type names are also bidirectionally converted (`str` <-> `string`).
6. **Petri net path resolution.** Declare "what you have" and "what you need," and the solver discovers composition paths.
7. **Optimization as a consequence of path discovery.** SIMD is not a cost function addition but the result of the instruction solver's BFS shortest-path discovery. Constant-time execution is the result of transition filtering. Parallelization is the result of automatic detection of independent transitions. Rather than adding optimization passes, Laplan leverages structural properties of the Petri net.
