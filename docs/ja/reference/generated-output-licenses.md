# 生成物ライセンス仕様

laplan が出力するファイルには、由来に応じて MIT または MPL-2.0 のライセンスヘッダが付与されます。この仕様は生成物のライセンス境界を定めるものであり、実際の生成物がこの記述と異なる場合はバグとして報告してください。

## 生成物のライセンス

| 出力の種類 | ライセンス | ヘッダ |
|---|---|---|
| 通常モジュール (型宣言、handler 骨格) | MIT | `SYNTHESIZED_HEADER` |
| runtime base (dispatch 骨格、boilerplate) | MIT | `SYNTHESIZED_HEADER` |
| WASM binding (TypeScript / Python / server) | MIT | `SYNTHESIZED_HEADER` |
| inverse 出力 (コード → `.lex` スケルトン) | MIT | `INVERSE_HEADER` |
| runtime solver (goal 合成、経路実行) | MPL-2.0 | runtime solver 専用ヘッダ |

### MIT と MPL-2.0 の境界

- `.lex` 宣言と mapping.lex テンプレートから決定論的に導かれる成果物は **MIT** です。利用者のデータの派生物であり、laplan 固有のアルゴリズムを含みません
- laplan 本体の solver ロジック (探索アルゴリズム、経路検証、dispatch 戦略) を生成物として含むファイルは **MPL-2.0** です

### 利用者にとっての意味

- MIT 出力は自由に取り込み、再配布時のライセンス制約は最小限です
- MPL-2.0 出力 (runtime solver) を改変した場合、そのファイルを MPL-2.0 で公開する義務があります。ただし周辺の MIT 出力や利用者自身のコードに MPL-2.0 が伝播することはありません (ファイル単位の copyleft)

## ヘッダの仕様

すべての生成物にはファイル先頭にライセンスヘッダが付与されます。ヘッダは 4 行構成で以下を含みます:

1. SPDX ライセンス識別子 (`SPDX-License-Identifier: MIT` または `MPL-2.0`)
2. Copyright 行
3. laplan 経由で生成された旨の注記
4. 改変と solver 互換性に関する注意

コメント接頭辞はターゲット言語に合わせて自動変換されます:

| 言語群 | 接頭辞 |
|---|---|
| Rust / C++ / Swift / TypeScript / Java 等 | `//` |
| Haskell / OCaml | `--` |
| Python / Ruby / Elixir | `#` |
| Erlang | `%%` |

接頭辞が変わってもヘッダ本文の意味は保持されます。

## 異常の判定基準

以下の状態は生成パイプラインの不備を示します。ヘッダの書き換えでは解決せず、生成物の出所の見直しが必要です。

| 状態 | 期待される正常動作 |
|---|---|
| MIT ヘッダのファイルに solver ロジックが含まれている | solver ロジックは MPL-2.0 ファイルにのみ含まれる |
| MPL-2.0 ヘッダのファイルが型宣言や decoder しか含まない | 型宣言と decoder は MIT ファイルに配置される |
| inverse 出力に MPL-2.0 ヘッダが付いている | inverse 出力は常に MIT (`INVERSE_HEADER`) |
| 生成物にライセンスヘッダが付いていない | すべての生成物にヘッダが付与される |
| `runtime.{ext}` (MIT) と `runtime_resolve.{ext}` (MPL-2.0) の境界が不明瞭 | ファイル名で MIT/MPL-2.0 が区別できる |

これらの状態を発見した場合は issue として報告してください。

---

## 設計の根拠

ここから先は境界の設計判断の根拠です。利用者がライセンスを判断するために読む必要はありません。

### 判断のフローチャート

生成物がどちらのライセンスに属するかは以下で判定されます:

1. その出力に laplan 本体の solver / dispatch / 推論ロジックが含まれるか → **MPL-2.0**
2. その出力が `.lex` + mapping.lex から決定論的に導けるか → **MIT**
3. 型 helper、encoder / decoder、定数宣言のみか → **MIT**
4. runtime dispatch の骨格 (boilerplate) のみで探索を含まないか → **MIT (runtime base)**
5. runtime が goal を受け取って経路を合成するか、経路検証を行うか → **MPL-2.0 (runtime solver)**

### ヘッダ定数の配置

`compiler/ir/src/lib.rs` に公開定数として定義され、各 emit ロジックが出力ファイル先頭に付与します。

| 定数 | 用途 | ライセンス |
|---|---|---|
| `SYNTHESIZED_HEADER` | synthesis 出力全般 | MIT |
| `INVERSE_HEADER` | inverse 出力 | MIT |

runtime solver 向けのヘッダは `atproto-server` feature 経由で `write_runtime_resolve` が emit し、MPL-2.0 を宣言します。

言語ごとのコメント接頭辞変換は `rewrite_header_prefix(header, prefix)` (`compiler/ir/src/lib.rs`) が担います。
