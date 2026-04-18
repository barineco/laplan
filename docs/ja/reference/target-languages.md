# 対応言語一覧

laplan は 21 言語への SDK synthesis をサポートします。言語テンプレートは `axiom/target/lang/{lang}/` に配置され、`GenericBackend` が統一駆動します。

## 全ターゲット

| 言語 | ディレクトリ | パス | カテゴリ |
|---|---|---|---|
| Rust | `rust/` | Lex₂ | システム |
| C++ | `cpp/` | Lex₂ | システム |
| D | `d/` | Lex₂ | システム |
| Zig | `zig/` | Lex₂ | システム |
| Go | `go/` | Lex₂ | システム |
| Swift | `swift/` | Lex₂ | モバイル |
| Kotlin | `kotlin/` | Lex₂ | JVM / モバイル |
| Java | `java/` | Lex₂ | JVM |
| Clojure | `clojure/` | Lex₂ | JVM / 関数型 |
| C# | `csharp/` | Lex₂ | .NET |
| TypeScript | `typescript/` | Lex₂ | Web |
| JavaScript | `javascript/` | Lex₂ | Web |
| Dart | `dart/` | Lex₂ | モバイル / Web |
| PHP | `php/` | Lex₂ | Web |
| Python | `python/` | Lex₂ | スクリプト |
| Ruby | `ruby/` | Lex₂ | スクリプト |
| Lua | `lua/` | Lex₂ | スクリプト |
| Haskell | `haskell/` | Lex₁ | 関数型 |
| OCaml | `ocaml/` | Lex₁ | 関数型 |
| Gleam | `gleam/` | Lex₁ | 関数型 |
| Elixir | `elixir/` | Lex₁ | 関数型 |

## Capability level

| Level | Capability | 名称 | 内容 |
|---|---|---|---|
| L1 | **type** | 型 | 型宣言のみ (product / sum / alias) |
| L2 | **interface** | API 構造 | handler trait + effect 型 |
| L3 | **recipe** | Recipe | recipe manifest + dispatch |
| L4 | **solver** | Goal synthesis | solver を呼んで経路合成 |

laplan の synthesis は L3 recipe までを安定供給し、L4 solver は言語ごとに順次対応中です。

## Lex₁ パス vs Lex₂ パス

| パス | 対象 | mapping 必須セクション |
|---|---|---|
| Lex₁ | Haskell, OCaml, Gleam, Elixir | `functional { let-in, match, lambda, fold }` |
| Lex₂ | 残り 17 言語 | `control`, `variable`, `handler` |

Lex₂ パスは `lowering.rs` が `FnExpr` → `Stmt` / `Expr` に自動降格します。Lex₁ パスは `template_engine_fn` が `FnExpr` を直接レンダリングします。

## 型対応表

各言語の `mapping.lex` → `syntax { product }` と `type_map` で定義されます。主要な型:

| lexicon 型 | Rust | TypeScript | Python | Haskell | Go |
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

正確な型は各 `axiom/target/lang/{lang}/mapping.lex` の `type_map` を参照してください。

## バインディング対象

WASM バイナリに対するバインディングは以下を自動生成します。

| 対象 | ファイル | feature |
|---|---|---|
| TypeScript | `bind_typescript.rs` | - |
| Python | `bind_python.rs` | - |

現在の bind 用テンプレートは `axiom/target/bind/wasmTypescript/` と `axiom/target/bind/wasmPython/` です。

## 追加のガイド

- 新言語の追加: [guide/adding-language.md](../guide/adding-language.md)
- synthesis パイプライン: [architecture/synthesis.md](../architecture/synthesis.md)
