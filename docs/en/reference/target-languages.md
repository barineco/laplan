# Supported Languages

laplan supports SDK synthesis for 21 languages. Language templates are placed under `axiom/target/lang/{lang}/` and driven uniformly by `GenericBackend`.

## All Targets

| Language | Directory | Path | Category |
|---|---|---|---|
| Rust | `rust/` | Lex₂ | System |
| C++ | `cpp/` | Lex₂ | System |
| D | `d/` | Lex₂ | System |
| Zig | `zig/` | Lex₂ | System |
| Go | `go/` | Lex₂ | System |
| Swift | `swift/` | Lex₂ | Mobile |
| Kotlin | `kotlin/` | Lex₂ | JVM / Mobile |
| Java | `java/` | Lex₂ | JVM |
| Clojure | `clojure/` | Lex₂ | JVM / Functional |
| C# | `csharp/` | Lex₂ | .NET |
| TypeScript | `typescript/` | Lex₂ | Web |
| JavaScript | `javascript/` | Lex₂ | Web |
| Dart | `dart/` | Lex₂ | Mobile / Web |
| PHP | `php/` | Lex₂ | Web |
| Python | `python/` | Lex₂ | Scripting |
| Ruby | `ruby/` | Lex₂ | Scripting |
| Lua | `lua/` | Lex₂ | Scripting |
| Haskell | `haskell/` | Lex₁ | Functional |
| OCaml | `ocaml/` | Lex₁ | Functional |
| Gleam | `gleam/` | Lex₁ | Functional |
| Elixir | `elixir/` | Lex₁ | Functional |

## Capability Levels

| Level | Capability | Name | Description |
|---|---|---|---|
| L1 | **type** | Types | Type declarations only (product / sum / alias) |
| L1-inv | **type-inverse** | Type inverse | Recovers product / sum / alias from generated code into `.lex` |
| L2 | **interface** | API structure | Handler trait + effect types |
| L3 | **recipe** | Recipe | Recipe manifest + dispatch |
| L4 | **solver** | Goal synthesis | Calls solver to synthesize paths |

laplan synthesis provides stable output through L3 recipe. L4 solver support coverage varies per language. L1-inv is supported for all 21 languages. `ast_inverse` auto-derives the inverse from the `syntax {}` section (product / sum / alias) and the pattern descriptions in the `expr {}` dual schema of mapping.lex, recovering type declarations without per-language implementation.

## Body Structuring Coverage

Body semantic extraction (`Stmt::Raw` fallback residual rate) during inverse conversion is supported as follows. The `ast_inverse` engine drives the `stmt {}` and `pattern {}` sections of mapping.lex and surfaces unstructurable fragments as `Stmt::Raw` / `Expr::Raw` / `Pattern::Raw`.

| Coverage | Languages | Notes |
|---|---|---|
| Full | Rust | Reference language. `stmt {}` and `pattern {}` sections are written in full; iterator idiom, if-let, match, and method chain are all structured. |
| Minimal | Other 20 languages | `stmt {}` covers `let-binding` / `return` / `assign-stmt` / `method-call-stmt` uniformly. Control-flow structuring is handled by the language-agnostic parser in engine. |
| Excluded | Clojure | S-expression macro-expanded form; outside the scope of body structuring and parity measurement. |

The 21-language aggregate body-structuring rate is monitored by `multilang_body_structuring.rs`. Residual rate decreases as each language's `stmt {}` / `pattern {}` sections are strengthened.

## Lex₁ Path vs Lex₂ Path

| Path | Target | Required mapping sections |
|---|---|---|
| Lex₁ | Haskell, OCaml, Gleam, Elixir | `functional { let-in, match, lambda, fold }` |
| Lex₂ | Remaining 17 languages | `control`, `variable`, `handler` |

The Lex₂ path uses `lowering.rs` to automatically lower `FnExpr` → `Stmt` / `Expr`. The Lex₁ path uses `template_engine_fn` to render `FnExpr` directly.

## Type Mapping Table

Defined in `syntax { product }` and `type_map` within each language's `mapping.lex`. Key types:

| lexicon type | Rust | TypeScript | Python | Haskell | Go |
|---|---|---|---|---|---|
| `string` | `String` | `string` | `str` | `Text` | `string` |
| `integer` | `i32` | `number` | `int` | `Int` | `int32` |
| `integer64` | `i64` | `bigint` | `int` | `Int` | `int64` |
| `float32` | `f32` | `number` | `float` | `Double` | `float32` |
| `number` (f64) | `f64` | `number` | `float` | `Double` | `float64` |
| `boolean` | `bool` | `boolean` | `bool` | `Bool` | `bool` |
| `bytes` | `Vec<u8>` | `Uint8Array` | `bytes` | `ByteString` | `[]byte` |
| `cid-link` | `Cid` | `CidLink` | `CidLink` | `CidLink` | `CidLink` |
| `blob` | `BlobRef` | `BlobRef` | `BlobRef` | `BlobRef` | `BlobRef` |

For exact types, refer to `type_map` in `axiom/target/lang/{lang}/mapping.lex` for each language.

## Binding Targets

The following bindings for WASM binaries are auto-generated.

| Target | File | Feature |
|---|---|---|
| TypeScript | `bind_typescript.rs` | - |
| Python | `bind_python.rs` | - |

The current binding templates are `axiom/target/bind/wasmTypescript/` and `axiom/target/bind/wasmPython/`.

## Additional Guides

- Adding a new language: [guide/adding-language.md](../guide/adding-language.md)
- Synthesis pipeline: [architecture/synthesis.md](../architecture/synthesis.md)
