# FAQ

## Big Picture

### What problem does Laplan solve? How is it different from existing SDK generators or OpenAPI?

OpenAPI and gRPC codegen are tools that "generate types and clients from a schema." The output is a calling skeleton; authentication, ordering, and error recovery are left to the user.

Laplan is a tool that "derives an executable plan, including paths, from type and relation declarations." By declaring inter-API dependencies (`requires` / `produces`) as transitions on a Petri net, the solver searches for reachable paths and automatically determines procedures including authentication, token refresh, and recovery. The user specifies only the desired result via `fetch_goal`.

### What does "metatype programming language" mean? How is it different from an ordinary typed language?

In an ordinary typed language, types classify values. `i32` is a set of integers, `String` is a set of strings.

In Laplan, types (places) are the starting point of computation. Declaring a type creates a Petri net, and the question "can I reach that type from this type?" becomes valid. It is a language that operates on types of types (metatypes): instead of classifying values, it declares and explores reachability between types.

### Why is a Petri net necessary? Wouldn't a simple dependency graph suffice?

A dependency graph (DAG) represents static relations like "A depends on B," but cannot handle token consumption and generation.

In API authentication flows, using a `refresh_jwt` consumes it and generates a new `access_jwt`. This "consumed on use" property (`consumes`) cannot be expressed in a DAG. Petri nets naturally model token consumption, generation, and exclusivity, and the solver accurately reflects these constraints when exploring reachability.

### What problems is Laplan good at? What is it not suited for?

**Strong at**: Flows spanning multiple APIs for authentication, data retrieval, and transformation. Batch generation of multi-language SDKs. Impact diagnosis when schemas change. Type-safe connections between client and server.

**Seemingly unsuited but actually expressible**: GUI event loops can be decomposed into per-event single-step transitions, naturally representable on a Petri net. Low-level memory operations are covered by `axiom/memory/`, which provides load/store type boundaries.

**Beyond the solver's search scope**: Numerical computations with unknown convergence (iterative convergence, open problems, etc.). These are addressed via [unsafe recursive and external bridges](architecture/solver.md). They can be described in Laplan, but the solver does not explore them as paths.

## Getting Started

### Should I read the README or docs/INDEX.md first?

The README is for understanding "what Laplan offers." INDEX.md is for finding the right document based on "what you want to do." Start with the README, then use the "task-based" table in INDEX.md.

### If I have existing code, should I start with invert or write .lex by hand?

It depends on the situation:

- **If you have AT Protocol Lexicon JSON**: The `.json ↔ .kdl ↔ .lex` fully reversible conversion is available, so enter directly from JSON
- **If you have existing Rust / TypeScript / etc. code**: Use `laplan invert` to generate a `.lex` skeleton, then add `rule` / `chain` on top
- **If designing from scratch**: The [getting-started](guide/getting-started.md) workflow of writing `.lex` by hand and letting the solver explore paths is recommended

## Formats and Declarations

### How far is the reversibility of the 3 formats (.json, .kdl, .lex)?

Declarations with the `xrpc=` attribute (AT Protocol-compatible endpoint definitions) are fully reversible across `.json ↔ .kdl ↔ .lex`. Type names are also bidirectionally converted (`str` ↔ `string`, `i32` ↔ `integer`).

`.lex`-specific declarations without `xrpc=` (rule, chain, law, derives, family, etc.) have no counterpart in Lexicon JSON and are not subject to `.lex` → `.json` conversion.

### What can .lex express that .json and .kdl cannot?

rule (reachability declarations), chain (manual path pinning), derives (derivation from existing declarations), law (algebraic laws), family (type families for isomorphic operations), inverse (inverse relations), dual (dual queries), and consumes (exclusive token consumption). See [layers.md](reference/layers.md) for details.

### What is the shortest way to understand the differences between type, lexicon, rule, func, and cratis?

| Concept | In one phrase | Petri net role |
|---|---|---|
| type | A kind of value | place (where tokens reside) |
| lexicon | API endpoint declaration | transition with type signature |
| rule | "what it requires and what it produces" | firing condition for a transition |
| func | function body | concrete type constraint for a transition |
| cratis | package | group of transitions. Declares interfaces via provides / requires |

## IR Layers

### How much should users be aware of Lex₀, Lex₁, Lex₂, Lex₃?

Lex₀ through Lex₃ are internal compiler classifications. Users need to be aware of top-level node types, not layer numbers:

| Explored by solver | Not explored but informs solver decisions | Meta / external connections |
|---|---|---|
| rule, const, assign, inverse, derives, handler/chain, refinement, lex | law, dual, invariant, family | import, cratis, meta |

The right column (law / dual / invariant / family) is not directly explored by the solver, but is practically important because it lets you express relationships without listing individual types. For example, `derives ... via batch` expresses the getProfile-to-getProfiles relationship in a single declaration, and family defines a closed set of types sharing isomorphic operations.

See [layers.md](reference/layers.md) for details.

### Why are nodes the solver does not explore (law, dual, family, etc.) important?

The solver explores transitions like rule / const / assign, but law / dual / invariant / family are not explored. However, they serve as the basis for solver pruning, identification, and derivation.

For example, declaring a family defines a closed set of types sharing isomorphic operations, and vectorize derives automatically generates SIMD transitions from it. Declaring a law lets the solver identify equivalent paths and reduce the search space. Declaring a dual expresses forward/reverse symmetry.

The same results could be achieved by listing individual rules, but the number of declarations would explode. The role of these nodes is to express structural relationships in a single declaration.

## Solver Operation

### How do I choose between rule and chain? At what point does chain become necessary?

rule declares "what is needed and what is produced," leaving path discovery to the solver. chain manually fixes "execute in this order."

chain becomes necessary when: (1) the solver returns multiple paths and constraint addition alone cannot make it unique, (2) the procedure is obvious and there is no benefit in having the solver search, (3) expressing the goal as types is difficult. Start with rule, try type refinement if MultiPath appears, and use chain if that still does not resolve it.

### How does goal-based operation differ from endpoint-based operation?

Goal-based is "I want profileViewDetailed." The solver discovers the path automatically.

Endpoint-based is "call getProfile." This is for cases where you pin ordering with chain or must route through a specific endpoint.

Goal-based is more declarative and leverages the solver's capabilities. Endpoint-based has higher compatibility with existing workflows.

### How should the 5 diagnosis levels be read in practice?

| Diagnosis | Reading | Next action |
|---|---|---|
| Matched | Success. Path found | Use as is |
| MultiPath | Multiple paths. Insufficient constraints | Add requires to rules or refine types |
| PrunedByBoundary | Blocked by auth / permissions / data | Check which of capability, ownership, or output is blocking |
| PrunedByRefinement | Blocked by a constraint added via refinement (e.g., auth tokens) | A path that would pass with bare Lexicon is blocked after adding capabilities. Supply the required auth information to the marking |
| MissingFact | Required fact is absent | Add produces to a rule or supply an external input |

### Is MultiPath an error or a design hint?

A design hint. It means "multiple paths at the same distance are indistinguishable," indicating that the declared constraints are insufficient. See [solver and type discipline](case/solver-type-discipline.md) for concrete resolution techniques.

### Can the shortest-path design really select the correct path?

The shortest path is "the solution with the fewest assumptions." Shorter paths require fewer `requires` to reach; longer paths demand more intermediate states. Path length serves as a measure of information content.

If the correct path is not the shortest, adding a concrete token as an intermediate `requires` naturally changes which path is shortest. Alternatively, chain can fix the path directly. In either case, no ambiguous state remains.

### When a path shorter than expected appears, should I suspect the solver or the type declarations?

The type declarations. The solver is faithful to the declared world and does not infer constraints beyond what is declared. A shorter-than-expected path (shortcut) almost always means type refinement is insufficient and intermediate states have been omitted. See [solver and type discipline](case/solver-type-discipline.md) for worked examples.

### When should semantic facts and completion tokens be introduced?

**Semantic facts**: When using a shared fact (`access_jwt`, etc.) directly as a goal causes unrelated endpoint rules to intrude. Replace with a fact that limits meaning to "the did produced in this endpoint's context."

**Completion tokens**: When confirmation that an endpoint has fully completed is needed. Setting the goal to a completion token (`create-account-complete`, etc.) prevents shortcuts that reach the goal through intermediate facts alone.

See [solver and type discipline](case/solver-type-discipline.md) for details.

### How do face, boundary, and trust="lexicon-only" work in distributed systems?

face switches "what to generate and what to trust from this perspective." In a client face, the server is treated as an axiom (trusted premise); in a server face, the reverse. The same declarations produce both client and server outputs.

`trust="lexicon-only"` is a mode that "trusts only the Lexicon type declarations of the counterpart face, without inspecting internal paths." The solver unconditionally accepts the endpoint's output type, and at runtime validates that the output matches the Lexicon type. This is a design for connecting across microservice boundaries by type boundary alone, without depending on the counterpart's internal implementation.

See [cratis](guide/cratis.md) for details.

## Languages and Generation

### How does resolver.lex differ from a normal .lex? Why does it need a dedicated format?

resolver.lex is a `.lex` file that describes the runtime solver's implementation using FnExpr (Lex₁ expression trees). Normal `.lex` files declare types and rules, but resolver.lex defines the solver's own behavior (path search, candidate classification, goal synthesis) as functions.

A dedicated format is necessary because the solver logic must be synthesized into all 21 languages. If implemented in Rust, it would be Rust-only; writing it in resolver.lex enables automatic generation for all 21 languages.

### With 21 languages supported, how practical is each language really?

Each language has a different maturity measured by capability level (L1-L4). See [target-languages.md](reference/target-languages.md) for details.

- L4 (runtime smoke test pass): Rust, Python, Go, Ruby, Kotlin, Java, Lua
- L3 (package build pass): TypeScript, Swift, C#, Dart, Zig, Clojure
- L2 (type compilation pass): Haskell, C++, D
- L1 (syntax lint pass): JavaScript, PHP, OCaml, Elixir, Gleam

### When adding a new language, in what scope is Rust-side implementation change truly unnecessary?

Adding only mapping.lex (type mapping), morph.lex (syntax patterns), and type.lex (type conversion) is sufficient when the language's syntax is brace-based (`{ }` to close blocks) or end-keyword-based (`def...end`).

Rust-side changes are needed when: (1) a new function extraction pattern is needed in inverse's `free_fn.rs` (e.g., Python's indent-based syntax), (2) the existing common parsers cannot cover the syntax structure. See [adding-language.md](guide/adding-language.md) for details.

### Why are bindings {}, package {}, and runtime-base {} separated into 3 layers?

The 3 layers have distinct responsibilities:

| Layer | Responsibility | Example |
|---|---|---|
| `bindings {}` | Connect axiom morphisms to external functions | `"signature.verify.ES256K"` → `k256::ecdsa::VerifyingKey::verify` |
| `package {}` | Generated package manifest template | `[dependencies] k256 = "0.13"` |
| `runtime-base {}` | Common types and helpers for generated code | `UnknownValue`, `Bytes`, `BlobRef` type definitions |

`bindings` names the function, `package` adds the dependency to the manifest so it compiles, and `runtime-base` provides the common types. Separating them allows different connection targets per language for the same axiom. See [axiom-bindings.md](guide/axiom-bindings.md) for details.

### Why is only the runtime solver MPL-2.0?

Normal generated outputs (type declarations, handler skeletons, decoders) are deterministically derived from the user's `.lex` declarations and mapping.lex templates, so they are MIT. They are derivatives of the user's data and contain no laplan-specific algorithms.

Runtime solver outputs contain laplan's search algorithms, path verification, and dispatch strategies, so they are MPL-2.0. MPL-2.0 is file-level copyleft: there is an obligation to publish modified solver files, but it does not propagate to surrounding MIT outputs or the user's own code. See [generated-output-licenses.md](reference/generated-output-licenses.md) for details.
