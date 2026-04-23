# 対応言語一覧

laplan は 21 言語への SDK synthesis をサポートします。言語テンプレートは `axiom/target/lang/{lang}/` に配置され、`GenericBackend` が統一駆動します。

## 全ターゲット

| 言語 | ディレクトリ | パス | カテゴリ | L1-inv |
|---|---|---|---|---|
| Rust | `rust/` | Lex₂ | システム | ok |
| C++ | `cpp/` | Lex₂ | システム | ok |
| D | `d/` | Lex₂ | システム | ok |
| Zig | `zig/` | Lex₂ | システム | ok |
| Go | `go/` | Lex₂ | システム | ok |
| Swift | `swift/` | Lex₂ | モバイル | ok |
| Kotlin | `kotlin/` | Lex₂ | JVM / モバイル | ok |
| Java | `java/` | Lex₂ | JVM | ok |
| Clojure | `clojure/` | Lex₂ | JVM / 関数型 | ok |
| C# | `csharp/` | Lex₂ | .NET | ok |
| TypeScript | `typescript/` | Lex₂ | Web | ok |
| JavaScript | `javascript/` | Lex₂ | Web | ok |
| Dart | `dart/` | Lex₂ | モバイル / Web | ok |
| PHP | `php/` | Lex₂ | Web | ok |
| Python | `python/` | Lex₂ | スクリプト | ok |
| Ruby | `ruby/` | Lex₂ | スクリプト | ok |
| Lua | `lua/` | Lex₂ | スクリプト | ok |
| Haskell | `haskell/` | Lex₁ | 関数型 | ok |
| OCaml | `ocaml/` | Lex₁ | 関数型 | ok |
| Gleam | `gleam/` | Lex₁ | 関数型 | ok |
| Elixir | `elixir/` | Lex₁ | 関数型 | ok |

## Capability level

| Level | Capability | 名称 | 内容 |
|---|---|---|---|
| L1 | **type** | 型 | 型宣言のみ (product / sum / alias) |
| L1-inv | **type-inverse** | 型逆変換 | 生成コードから product / sum / alias を `.lex` に復元 |
| L2 | **interface** | API 構造 | handler trait + effect 型 |
| L3 | **recipe** | Recipe | recipe manifest + dispatch |
| L4 | **solver** | Goal synthesis | solver を呼んで経路合成 |

laplan の synthesis は L3 recipe までを安定供給し、L4 solver は言語ごとに順次対応中です。L1-inv は全 21 言語が対応しています。mapping.lex の `syntax {}` セクション (product / sum / alias) と `expr {}` dual schema の pattern 記述から `ast_inverse` が自動導出することで、言語ごとの個別実装なしに型宣言の復元が可能です。

## L3 recipe manifest の実装状況 (2026-04-20 時点)

各言語の `mapping.lex` における `runtime-solve-*` キー (recipe manifest + dispatch 生成) の実装状況。L3 capability 判定の基礎情報です。

| runtime-solve-* キー数 | 言語 |
|---:|---|
| 4 | java |
| 3 | cpp, csharp, go, haskell, lua |
| 2 | clojure, d, elixir, kotlin, php, rust |
| 1 | dart, gleam, ocaml, ruby, swift, zig |
| 0 | javascript, python, typescript (WASM binding 経路で補完する設計) |

`runtime-solve-filename` / `runtime-solve-header` / `runtime-solve-module-name` 等のキーが該当。キー数ゼロの 3 言語 (javascript / python / typescript) は現状 recipe manifest 生成未対応で、Inv-4 L3 level (統合 spec 参照) の passing は 0 のまま。WASM binding 経路 (`axiom/target/bind/wasmPython`, `axiom/target/bind/wasmTypescript`) で補完する設計は別途検討中。

## bindings セクションの実装状況

`bindings {}` セクション (axiom の射を外部 crate / 関数に接続) の実装状況:

| bindings 実装状況 | 言語 |
|---|---|
| 実装あり (20 言語) | clojure, cpp, csharp, d, dart, elixir, gleam, go, haskell, java, javascript, kotlin, lua, ocaml, php, python, ruby, rust, swift, typescript |
| 未実装 (1 言語) | **zig** |

zig のみ `bindings {}` セクション未実装で、axiom の外部接続経路が未整備です。21 言語全件で bindings を整備することが Inv-5 (dual schema 対称性) の前提になります (統合 spec 参照)。

## body 構造化対応

逆変換時の body 意味論抽出 (`Stmt::Raw` fallback 残存率) の対応状況は以下の通りです。`ast_inverse` engine は mapping.lex の `stmt {}` / `pattern {}` セクションを駆動し、構造化できない部分を `Stmt::Raw` / `Expr::Raw` / `Pattern::Raw` として可視化します。

| 対応度 | 言語 | 説明 |
|---|---|---|
| 完全 | Rust | reference 言語。`stmt {}` / `pattern {}` セクションをフル記述、iterator idiom / if-let / match / method chain を構造化 |
| 最小カバー | 他 20 言語 | `stmt {}` に `let-binding` / `return` / `assign-stmt` / `method-call-stmt` を横断整備。制御構造は engine の言語非依存 parser が担当 |
| 計測なし | Clojure | S 式 macro 展開形のため body 構造化と parity 計算を適用しない |

21 言語合算の body 構造化率は `multilang_body_structuring.rs` が監視します。各言語 mapping.lex の `stmt {}` / `pattern {}` セクションを補強することで残存率を下げられます。

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
