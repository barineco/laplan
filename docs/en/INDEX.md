# laplan Documentation INDEX

laplan is a Lexicon-based programmable language platform. Declare types and morphisms in KDL, and the Petri net solver finds synthesis paths and generates SDKs for 21 languages plus WASM binaries.

## Document List

### Architecture

| File | Description |
|---|---|
| [architecture/overview.md](architecture/overview.md) | Crate dependency graph, full pipeline overview, position of the two-layer solver |
| [architecture/parser.md](architecture/parser.md) | `compiler/kdl` + `compiler/ir` parser layer. KDL → AST → IR conversion |
| [architecture/ir.md](architecture/ir.md) | Lex₁/Lex₂ intermediate representation, FnExpr / Stmt / Expr, lowering |
| [architecture/compiler.md](architecture/compiler.md) | WASM binary generation via `compiler/compile`, inverse functor via `compiler/inverse` |
| [architecture/synthesis.md](architecture/synthesis.md) | Multi-language code generation in `compiler/synthesis`, mapping.lex / morph.lex / type.lex |
| [architecture/solver.md](architecture/solver.md) | Petri net solver, two-layer structure (morphism / instruction), pruning |
| [architecture/cli.md](architecture/cli.md) | Subcommands and options for the `laplan` binary |

### Guides

| File | Description |
|---|---|
| [guide/getting-started.md](guide/getting-started.md) | Installation, first `.lex`, lint / solve / synthesis workflow |
| [guide/cratis.md](guide/cratis.md) | Writing cratis, provides/requires, workspace cratis, faces |
| [guide/adding-language.md](guide/adding-language.md) | Steps to add a new language under `axiom/target/lang/` |
| [guide/wasm-extension.md](guide/wasm-extension.md) | Building the VSCode extension and WASM, Petri net webview |

### Reference

| File | Description |
|---|---|
| [reference/layers.md](reference/layers.md) | `.lex` top nodes and Lex₀/₁/₂/₃ layer classification |
| [reference/axiom-numeric.md](reference/axiom-numeric.md) | `axiom/i32` `axiom/i64` `axiom/f32` `axiom/f64` `axiom/bool` numeric primitives |
| [reference/axiom-string.md](reference/axiom-string.md) | `axiom/str` `axiom/bytes` strings and byte sequences |
| [reference/axiom-serialization.md](reference/axiom-serialization.md) | `axiom/json` `axiom/cbor` `axiom/kdl` serialization |
| [reference/axiom-content-addressing.md](reference/axiom-content-addressing.md) | `axiom/cid` `axiom/car` content addressing |
| [reference/axiom-crypto.md](reference/axiom-crypto.md) | `axiom/crypto` hashing, signing, key derivation |
| [reference/axiom-algebra.md](reference/axiom-algebra.md) | `axiom/algebra` algebraic structures and family (product, vectorize) |
| [reference/axiom-category.md](reference/axiom-category.md) | `axiom/category` category theory primitives (compose, dual, lift) |
| [reference/axiom-memory.md](reference/axiom-memory.md) | `axiom/memory` memory model |
| [reference/generated-output-licenses.md](reference/generated-output-licenses.md) | MIT / MPL-2.0 boundaries per generated output |
| [reference/target-languages.md](reference/target-languages.md) | Capability levels, type mapping tables, and category classification for 21 languages |

### Case Studies

| File | Description |
|---|---|
| [case/solver-type-discipline.md](case/solver-type-discipline.md) | Eliminating solver shortcuts through type refinement. Storage evidence types, semantic facts, completion tokens |

## By Task

### Read and Understand

| Task | Documentation |
|---|---|
| Get the overall picture | [overview](architecture/overview.md), [layers](reference/layers.md) |
| Check how to write `.lex` | [layers](reference/layers.md), [getting-started](guide/getting-started.md) |
| Trace the pipeline | [overview](architecture/overview.md), [parser](architecture/parser.md), [ir](architecture/ir.md) |
| Check license boundaries for generated outputs | [generated-output-licenses](reference/generated-output-licenses.md) |
| Understand how the solver works | [solver](architecture/solver.md) |
| Eliminate solver shortcuts | [solver-type-discipline](case/solver-type-discipline.md), [solver](architecture/solver.md) |
| Understand how synthesis works | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md) |
| Understand the Lex1 path (for functional languages) | [synthesis](architecture/synthesis.md), [ir](architecture/ir.md) |
| Understand WASM binary generation | [compiler](architecture/compiler.md) |
| Check the coverage of the inverse functor (code → `.lex`) | [compiler](architecture/compiler.md) ("laplan-inverse" section) |
| Check supported languages and their maturity | [target-languages](reference/target-languages.md) |

### Operate

| Task | Documentation |
|---|---|
| Write the first `.lex` and solve | [getting-started](guide/getting-started.md), [cli](architecture/cli.md) |
| Use lint / solve commands | [cli](architecture/cli.md) |
| Define a cratis | [cratis](guide/cratis.md), [layers](reference/layers.md) |
| Check which cratis an axiom belongs to | [cratis](guide/cratis.md), [axiom-crypto](reference/axiom-crypto.md), [axiom-algebra](reference/axiom-algebra.md) |
| Add an operation to an axiom | [ir](architecture/ir.md), [synthesis](architecture/synthesis.md), [axiom-algebra](reference/axiom-algebra.md) |
| Add a new language | [adding-language](guide/adding-language.md), [synthesis](architecture/synthesis.md) |
| Work with the VSCode extension | [wasm-extension](guide/wasm-extension.md) |

### Modify

| Task | Documentation |
|---|---|
| Edit resolver.lex | [ir](architecture/ir.md) ("resolver.lex" section), [layers](reference/layers.md) |
| Change IR types (FnExpr / Stmt / Expr) | [ir](architecture/ir.md) |
| Change lowering transformations | [ir](architecture/ir.md), [synthesis](architecture/synthesis.md) |
| Add mapping.lex template variables | [synthesis](architecture/synthesis.md), [adding-language](guide/adding-language.md) |
| Improve solver pruning | [solver](architecture/solver.md) |
| Refine rule requires/produces | [solver-type-discipline](case/solver-type-discipline.md) |
| Change WASM generation optimizations | [compiler](architecture/compiler.md) |

## Structure Mapping

### workspace member → docs

| Member | Primary References |
|---|---|
| `compiler/kdl` (laplan-kdl) | [parser](architecture/parser.md) |
| `compiler/ir` (laplan-ir) | [ir](architecture/ir.md), [parser](architecture/parser.md) |
| `compiler/compile` (laplan-compile) | [compiler](architecture/compiler.md), [solver](architecture/solver.md) |
| `compiler/synthesis` (laplan-synthesis) | [synthesis](architecture/synthesis.md) |
| `compiler/inverse` (laplan-inverse) | [compiler](architecture/compiler.md) |
| `compiler/cli` (laplan) | [cli](architecture/cli.md) |
| `extension/` (VSCode extension) | [wasm-extension](guide/wasm-extension.md) |
| `vendored-json/` | [parser](architecture/parser.md) |

### axiom → docs

| Directory | Primary References |
|---|---|
| `axiom/i32`, `axiom/i64`, `axiom/f32`, `axiom/f64`, `axiom/bool` | [axiom-numeric](reference/axiom-numeric.md) |
| `axiom/str`, `axiom/bytes` | [axiom-string](reference/axiom-string.md) |
| `axiom/json`, `axiom/cbor`, `axiom/kdl` | [axiom-serialization](reference/axiom-serialization.md) |
| `axiom/cid`, `axiom/car` | [axiom-content-addressing](reference/axiom-content-addressing.md) |
| `axiom/crypto` | [axiom-crypto](reference/axiom-crypto.md) |
| `axiom/algebra` | [axiom-algebra](reference/axiom-algebra.md) |
| `axiom/category` | [axiom-category](reference/axiom-category.md) |
| `axiom/memory` | [axiom-memory](reference/axiom-memory.md) |
| `axiom/resolver.lex` | [ir](architecture/ir.md) ("resolver.lex" section), [layers](reference/layers.md) |
| `axiom/target/lang` | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md), [adding-language](guide/adding-language.md) |
| `axiom/target/bind` | [synthesis](architecture/synthesis.md) |
| `axiom/target/binary` | [compiler](architecture/compiler.md) |
| `axiom/solve` | [solver](architecture/solver.md) |

### Change Area → Impact Scope

An index for identifying which docs and downstream projects (such as neco-atproto) are affected when making changes to laplan.

| Change Area | laplan docs | Downstream docs |
|---|---|---|
| axiom/ (adding types, morphisms, constraints) | Relevant [axiom-*](reference/axiom-algebra.md) files | neco-atproto `guide/codegen.md`, `architecture/crate-map.md` |
| axiom/target/lang/ (mapping changes) | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md) | neco-atproto `guide/{lang}.md` |
| solver (search improvements) | [solver](architecture/solver.md) | neco-atproto `protocol/04-lexicon.md`, `architecture/overview.md` |
| synthesis output format | [synthesis](architecture/synthesis.md) | neco-atproto all `guide/{lang}.md` |
| inverse (functor) coverage | [compiler](architecture/compiler.md), [cli](architecture/cli.md) | neco-atproto `development/` inverse conversion verification |
| cratis structure | [cratis](guide/cratis.md), [layers](reference/layers.md) | neco-atproto `protocol/04-lexicon.md`, `architecture/overview.md` |
| CLI subcommands | [cli](architecture/cli.md) | neco-atproto `guide/codegen.md` |
