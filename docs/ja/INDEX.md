# laplan ドキュメント INDEX

laplan は Lexicon ベースのプログラマブル言語基盤です。型と射を KDL で宣言すると、Petri net solver が合成経路を探し、21 言語の SDK + WASM バイナリを導出します。

## ドキュメント一覧

### アーキテクチャ

| ファイル | 内容 |
|---|---|
| [architecture/overview.md](architecture/overview.md) | crate 依存グラフ、パイプライン全体像、二層 solver の位置づけ |
| [architecture/parser.md](architecture/parser.md) | `compiler/kdl` + `compiler/ir` パーサ層。KDL → AST → IR の変換 |
| [architecture/ir.md](architecture/ir.md) | Lex₁/Lex₂ 中間表現、FnExpr / Stmt / Expr、lowering |
| [architecture/compiler.md](architecture/compiler.md) | `compiler/compile` による WASM バイナリ生成、`compiler/inverse` による逆関手 |
| [architecture/synthesis.md](architecture/synthesis.md) | `compiler/synthesis` の多言語コード生成、mapping.lex / morph.lex / type.lex |
| [architecture/solver.md](architecture/solver.md) | Petri net solver、二層構成 (morphism / instruction)、枝刈り |
| [architecture/cli.md](architecture/cli.md) | `laplan` バイナリのサブコマンド一覧とオプション |

### ガイド

| ファイル | 内容 |
|---|---|
| [guide/getting-started.md](guide/getting-started.md) | インストール、最初の `.lex`、lint / solve / synthesis のワークフロー |
| [guide/cratis.md](guide/cratis.md) | cratis の書き方、provides/requires、workspace cratis、faces |
| [guide/adding-language.md](guide/adding-language.md) | `axiom/target/lang/` に新言語を追加する手順 |
| [guide/axiom-bindings.md](guide/axiom-bindings.md) | axiom 実装の言語別差し替え (`bindings {}` セクション) |
| [guide/wasm-extension.md](guide/wasm-extension.md) | VSCode 拡張と WASM のビルド、Petri net webview |

### リファレンス

| ファイル | 内容 |
|---|---|
| [reference/layers.md](reference/layers.md) | `.lex` トップノードと Lex₀/₁/₂/₃ 層分類 |
| [reference/axiom-numeric.md](reference/axiom-numeric.md) | `axiom/i32` `axiom/i64` `axiom/f32` `axiom/f64` `axiom/bool` 数値プリミティブ |
| [reference/axiom-string.md](reference/axiom-string.md) | `axiom/str` `axiom/bytes` 文字列・バイト列 |
| [reference/axiom-serialization.md](reference/axiom-serialization.md) | `axiom/json` `axiom/cbor` `axiom/kdl` シリアライゼーション |
| [reference/axiom-content-addressing.md](reference/axiom-content-addressing.md) | `axiom/cid` `axiom/car` コンテンツアドレッシング |
| [reference/axiom-crypto.md](reference/axiom-crypto.md) | `axiom/crypto` ハッシュ・署名・鍵導出 |
| [reference/axiom-algebra.md](reference/axiom-algebra.md) | `axiom/algebra` 代数構造と family (product, vectorize) |
| [reference/axiom-category.md](reference/axiom-category.md) | `axiom/category` 圏論プリミティブ (compose, dual, lift) |
| [reference/axiom-memory.md](reference/axiom-memory.md) | `axiom/memory` メモリモデル |
| [reference/generated-output-licenses.md](reference/generated-output-licenses.md) | 生成物ごとの MIT / MPL-2.0 境界 |
| [reference/target-languages.md](reference/target-languages.md) | 21 言語の Capability level、型対応表、カテゴリ分類 |

### ケーススタディ

| ファイル | 内容 |
|---|---|
| [case/solver-type-discipline.md](case/solver-type-discipline.md) | solver のショートカットを型の精密化で解消する手法。格納証拠型、semantic fact、completion token |

## やりたいこと別

### 読む・理解する

| やりたいこと | 読むドキュメント |
|---|---|
| 全体像を掴む | [overview](architecture/overview.md), [layers](reference/layers.md) |
| `.lex` の書き方を確認する | [layers](reference/layers.md), [getting-started](guide/getting-started.md) |
| パイプラインを追跡する | [overview](architecture/overview.md), [parser](architecture/parser.md), [ir](architecture/ir.md) |
| 生成物のライセンス境界を確認する | [generated-output-licenses](reference/generated-output-licenses.md) |
| solver の仕組みを理解する | [solver](architecture/solver.md) |
| solver のショートカットを解消する | [solver-type-discipline](case/solver-type-discipline.md), [solver](architecture/solver.md) |
| synthesis の仕組みを理解する | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md) |
| Lex1 パス (関数型言語向け) を理解する | [synthesis](architecture/synthesis.md), [ir](architecture/ir.md) |
| WASM バイナリ生成の流れを知る | [compiler](architecture/compiler.md) |
| 逆関手 (コード → `.lex`) の対応範囲を確認する | [compiler](architecture/compiler.md) (「laplan-inverse」節) |
| 対応言語とその成熟度を確認する | [target-languages](reference/target-languages.md) |

### 操作する

| やりたいこと | 読むドキュメント |
|---|---|
| 最初の `.lex` を書いて solve する | [getting-started](guide/getting-started.md), [cli](architecture/cli.md) |
| lint / solve コマンドを使う | [cli](architecture/cli.md) |
| cratis を定義する | [cratis](guide/cratis.md), [layers](reference/layers.md) |
| axiom の所属 cratis を確認する | [cratis](guide/cratis.md), [axiom-crypto](reference/axiom-crypto.md), [axiom-algebra](reference/axiom-algebra.md) |
| axiom に演算を追加する | [ir](architecture/ir.md), [synthesis](architecture/synthesis.md), [axiom-algebra](reference/axiom-algebra.md) |
| axiom の実装を別パッケージに差し替える | [axiom-bindings](guide/axiom-bindings.md) |
| 新言語を追加する | [adding-language](guide/adding-language.md), [synthesis](architecture/synthesis.md) |
| VSCode 拡張を触る | [wasm-extension](guide/wasm-extension.md) |

### 変更する

| やりたいこと | 読むドキュメント |
|---|---|
| resolver.lex を編集する | [ir](architecture/ir.md) (「resolver.lex」節), [layers](reference/layers.md) |
| IR 型 (FnExpr / Stmt / Expr) を変える | [ir](architecture/ir.md) |
| lowering 変換を変える | [ir](architecture/ir.md), [synthesis](architecture/synthesis.md) |
| mapping.lex テンプレート変数を増やす | [synthesis](architecture/synthesis.md), [adding-language](guide/adding-language.md) |
| solver の枝刈りを改善する | [solver](architecture/solver.md) |
| rule の requires/produces を精密化する | [solver-type-discipline](case/solver-type-discipline.md) |
| WASM 生成の最適化を変える | [compiler](architecture/compiler.md) |

## 構造マッピング

### workspace member → docs

| member | 主な参照先 |
|---|---|
| `compiler/kdl` (laplan-kdl) | [parser](architecture/parser.md) |
| `compiler/ir` (laplan-ir) | [ir](architecture/ir.md), [parser](architecture/parser.md) |
| `compiler/compile` (laplan-compile) | [compiler](architecture/compiler.md), [solver](architecture/solver.md) |
| `compiler/synthesis` (laplan-synthesis) | [synthesis](architecture/synthesis.md) |
| `compiler/inverse` (laplan-inverse) | [compiler](architecture/compiler.md) |
| `compiler/cli` (laplan) | [cli](architecture/cli.md) |
| `extension/` (VSCode 拡張) | [wasm-extension](guide/wasm-extension.md) |
| `vendored-json/` | [parser](architecture/parser.md) |

### axiom → docs

| ディレクトリ | 主な参照先 |
|---|---|
| `axiom/i32`, `axiom/i64`, `axiom/f32`, `axiom/f64`, `axiom/bool` | [axiom-numeric](reference/axiom-numeric.md) |
| `axiom/str`, `axiom/bytes` | [axiom-string](reference/axiom-string.md) |
| `axiom/json`, `axiom/cbor`, `axiom/kdl` | [axiom-serialization](reference/axiom-serialization.md) |
| `axiom/cid`, `axiom/car` | [axiom-content-addressing](reference/axiom-content-addressing.md) |
| `axiom/crypto` | [axiom-crypto](reference/axiom-crypto.md) |
| `axiom/algebra` | [axiom-algebra](reference/axiom-algebra.md) |
| `axiom/category` | [axiom-category](reference/axiom-category.md) |
| `axiom/memory` | [axiom-memory](reference/axiom-memory.md) |
| `axiom/resolver.lex` | [ir](architecture/ir.md) (「resolver.lex」節), [layers](reference/layers.md) |
| `axiom/target/lang` | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md), [adding-language](guide/adding-language.md) |
| `axiom/target/bind` | [synthesis](architecture/synthesis.md) |
| `axiom/target/binary` | [compiler](architecture/compiler.md) |
| `axiom/solve` | [solver](architecture/solver.md) |

### 変更領域 → 影響範囲

laplan に変更を入れる際、影響する docs と下流プロジェクト (neco-atproto 等) を把握する索引です。

| 変更領域 | laplan docs | 下流 docs |
|---|---|---|
| axiom/ (型・射・制約の追加) | [axiom-*](reference/axiom-algebra.md) の該当ファイル | neco-atproto `guide/codegen.md`, `architecture/crate-map.md` |
| axiom/target/lang/ (mapping 変更) | [synthesis](architecture/synthesis.md), [target-languages](reference/target-languages.md) | neco-atproto `guide/{lang}.md` |
| solver (探索改善) | [solver](architecture/solver.md) | neco-atproto `protocol/04-lexicon.md`, `architecture/overview.md` |
| synthesis 出力形式 | [synthesis](architecture/synthesis.md) | neco-atproto 全 `guide/{lang}.md` |
| inverse (逆関手) の対応範囲 | [compiler](architecture/compiler.md), [cli](architecture/cli.md) | neco-atproto `development/` 逆変換検証 |
| cratis 構造 | [cratis](guide/cratis.md), [layers](reference/layers.md) | neco-atproto `protocol/04-lexicon.md`, `architecture/overview.md` |
| CLI サブコマンド | [cli](architecture/cli.md) | neco-atproto `guide/codegen.md` |
