# Laplan: Lexicon as Programming Language

[日本語](README-ja.md) | [docs](docs/en/INDEX.md)

A metatype programming language that declares types and relations in KDL format and synthesizes them via Petri net path resolution.

## What You Have, What You Want

An API takes an input and returns an output. This input-to-output relation is a type mapping. getProfile takes an actor and returns a profile. createSession takes a credential and returns an access_jwt.

Chain these mappings together, and a path emerges from "what you have" to "what you want." Laplan automates this path discovery.

```rust
// Traditional: manually write auth → request → error handling → retry
let session = client.create_session(&credentials).await?;
let did = client.resolve_handle(&session.access_jwt, &handle).await?;
let profile = client.get_profile(&session.access_jwt, &did).await?;
// access_jwt expired? → manually refresh and retry...
```

```rust
// Laplan: say what you want
client.fetch_goal("datum:profileViewDetailed", &params)
```

One line. The solver automatically discovers and executes all steps: authentication, token refresh, handle resolution, and request dispatch.

All you need to declare is what each API requires and what it produces:

```kdl
rule "com.atproto.server.createSession" {
    produces capability="access_jwt"
    produces capability="refresh_jwt" ttl=86400
}

rule "app.bsky.actor.getProfile" {
    requires capability="access_jwt"
    requires input="actor"
    produces output="profile"
}
```

The solver reads these declarations as a [Petri net](https://en.wikipedia.org/wiki/Petri_net) and discovers reachable paths via BFS (breadth-first search). No procedures to write. If you want explicit ordering, use a chain.

---

## Why This Matters

### Bugs That Cannot Exist

There is no implicit procedure ordering. "Sending a request before authenticating," "forgetting to refresh a token," "calling APIs in the wrong order" cannot happen because the solver determines the path automatically. Ordering bugs vanish when the ordering is derived, not authored.

### Never Send Unreachable Requests

The solver computes reachability before execution. If "this request is unreachable because there's no auth token" is determined, no HTTP request is sent. Running the solver on a connected client-server system means unreachable requests aren't even attempted, reducing network load and making the system naturally suited for distributed architectures.

When unreachable, the solver [classifies what's missing](docs/en/architecture/solver.md) into 5 levels:

| Diagnosis | Meaning |
|---|---|
| **Matched** | Reachable, path found |
| **MultiPath** | Multiple paths found (signals insufficient constraints) |
| **PrunedByBoundary** | Blocked by auth / permissions / insufficient data |
| **PrunedByRefinement** | Blocked by constraints added via refinement (e.g., auth tokens) |
| **MissingFact** | Required fact does not exist |

### Dramatically Shorter Code

Write only declarations. Procedures, error handling, retry logic, and authentication flows are all derived from the solver's path discovery. You may write chains, but you don't have to.

### Semantic-Preserving Transformation Across 21 Languages

Laplan has bidirectional transformation for 21 languages:

- **[synthesis](docs/en/architecture/synthesis.md)**: Lex IR → source code generation for 21 languages
- **[inverse](docs/en/architecture/compiler.md)**: source code from 21 languages → Lex IR

Combining these:

```
Source(Rust) ──inverse──→ Lex IR ──synthesis──→ Source(TypeScript)
```

Enter from existing code and convert to another language. Traditional transpilers (C2Rust, j2objc, etc.) require a dedicated converter per language pair (N^2 structure). Laplan uses Lex IR as a hub (N×2 structure). One mapping.lex file adds one language.

The correctness criterion is not syntactic similarity but **alpha-equivalence** (logically identical up to bound variable renaming). This is validated through roundtrip tests across 21 languages × 9 functions.

| Language | Category | | Language | Category |
|---|---|---|---|---|
| Rust | Systems | | Swift | Mobile |
| C++ | Systems | | Dart | Mobile |
| Zig | Systems | | Kotlin | JVM / Mobile |
| Go | Systems | | Java | JVM |
| D | Systems | | C# | .NET |
| TypeScript | Web | | Clojure | JVM |
| JavaScript | Web | | Haskell | Functional |
| Python | Scripting | | OCaml | Functional |
| Ruby | Scripting | | Elixir | Functional |
| PHP | Scripting | | Gleam | Functional |
| Lua | Scripting | | | |

### Automatic Dependency Resolution

Multiple language implementations can be connected to the same capability:

```kdl
import "neco-vault" {
    procedure "verify_signature" {
        in { (bytes)message; (bytes)signature; (bytes)pubkey }
        out { (bool)valid }
    }
}
```

When emitting to TypeScript, only the npm package dependency remains. When emitting to Rust, only the neco-crates dependency remains. The solver automatically prunes unreachable dependencies. No need to manually write import tables per language pair.

### Schema Evolution Diagnosis

When an API changes, the solver immediately reports which paths broke. Unreachable transitions and newly required capabilities are detected automatically. No need to grep for call sites.

### Automatic Recovery Paths

When `access_jwt` expires, the solver works backwards from "what's missing" and automatically inserts the `refreshSession` path. No retry logic to author.

### Automatic SIMD / GPU / Parallelization

The solver searches SIMD and scalar paths in the same table, automatically selecting SIMD as the shortest path. This is not an optimization pass but a consequence of shortest-path discovery on the Petri net. See [two-layer solver](docs/en/architecture/solver.md) for details.

```bash
laplan compile cratis/encrypted-dm/ --bake --simd --constant-time
```

`--bake` (enumerate all paths ahead of time), `--simd` (auto-select SIMD), `--constant-time` (branchless paths only) are orthogonal flags, freely combinable.

### Formally Verified Soundness

Solver reachability, constraint-dependent reachability changes, and non-recoverability of annotations from types are formally verified in [lean-lexicon](https://github.com/barineco/lean-lexicon) using Lean 4 / Mathlib (zero sorry).

---

## Declaration Structure

Constraints are [layered across 4 levels](docs/en/reference/layers.md). Higher levels delegate to the solver; lower levels give the programmer control. **If higher levels suffice, lower ones can be omitted.**

| Layer | Declares | Style |
|---|---|---|
| **type** (place) | What exists | Most declarative |
| **rule** (transition) | What can reach what | |
| **function** | What goes in and out | |
| **chain** | In what order | Most imperative (optional) |

Write types and the solver finds paths. If insufficient, add rules for direction, functions for input/output, chains for ordering. At high levels (APIs), rules alone often suffice.

Details: [docs/en/INDEX.md](docs/en/INDEX.md)

---

## Three Formats

The same API definition (`app.bsky.actor.getProfile`) can be written in 3 formats:

| Format | Purpose | Interconversion |
|---|---|---|
| `.json` (Lexicon JSON) | AT Protocol canonical source | `.json ↔ .kdl ↔ .lex` fully reversible |
| `.kdl` (Lexicon KDL) | Human-readable projection of JSON | |
| `.lex` (Laplan KDL) | Computational language. Can express rule / chain / law | |

There is a proven track record of loading AT Protocol Lexicons (~100 NSIDs) as-is, adding just a few lines of capability for request headers, and having the solver automatically resolve all paths.

---

## Tooling

| Tool | Status | Description |
|---|---|---|
| `laplan` CLI | Available | compile, synthesis, inverse, solve, diagnose |
| VSCode extension (linter) | Available | Real-time diagnostics for `.lex` files |
| Graph visualization | In development | Node editor for Petri net / dependency graphs |
| WASM package | Available | Run the solver in browser / Node.js |

---

## Solver Design Decisions

### Why Shortest Path

The solver returns the shortest path via BFS. This is a semantic choice, not an implementation convenience.

A shorter path requires fewer `requires` to reach. The shortest path is the solution with the fewest assumptions. Conversely, a longer path passes through more intermediate states and demands more specific constraints. Path length serves as a measure of information content.

When constraints are insufficient, the solver returns multiple paths at the same distance ([MultiPath diagnosis](docs/en/architecture/solver.md)). This is design feedback: "the shortest paths are indistinguishable." To make the path unique, add more concrete and semantic constraints. See [solver and type discipline](docs/en/case/solver-type-discipline.md) for worked examples of resolving ambiguity through type refinement.

- **Add an intermediate token**: inserting a `requires` causes the correct shortest path to emerge naturally
- **Pin with chain**: specify ordering directly, bypassing the solver

Neither leaves an ambiguous state. The former changes which path is shortest; the latter fixes the path manually.

### Termination and Computational Boundaries

The solver always terminates. Each rule is a single transition, and iteration is expressed through fold and [safe recursive](docs/en/reference/axiom/algebra.md) (bounded / decreasing). This suffices for most practical use cases.

For computations with unknown convergence (iterative numerical methods, open problems, etc.), two mechanisms are available:

| Mechanism | Solver treatment | Use case |
|---|---|---|
| unsafe recursive | Recognized as a label but not explored as a path (Lex₂) | Iteration with non-obvious convergence |
| External bridge (import) | Only the type boundary is declared; internal implementation is trusted | Delegation to external libraries |

Larger systems asymptotically approach Turing completeness, but the space the solver explores is always finite and halts. See [solver](docs/en/architecture/solver.md) for details.

---

## Design Principles

1. **Types are the foundation.** If types exist, a Petri net emerges and path questions become valid.
2. **Declarations are the source of truth.** Code is generated from declarations. Manually authored code implies insufficient declarations.
3. **Optimization is a consequence of path discovery.** SIMD, constant-time execution, and parallelization are derived from structural properties of the Petri net.

---

Declaration syntax, architecture, axiom reference, and language addition guides are in [docs](docs/en/INDEX.md).
