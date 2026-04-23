# FAQ

## 全体像

### Laplan は何を解く道具ですか。既存の SDK 生成や OpenAPI と何が違いますか。

OpenAPI や gRPC codegen は「スキーマから型とクライアントを生成する」道具です。生成物は呼び出しの骨格であり、認証・順序制御・エラー回復は利用者が手で書きます。

Laplan は「型と関係の宣言から、経路を含む実行可能な計画を導出する」道具です。API 間の依存 (`requires` / `produces`) を Petri net 上の遷移として宣言すると、solver が到達可能な経路を探索し、認証・トークン更新・リカバリを含む手順を自動決定します。利用者は `fetch_goal` で欲しい結果を指定するだけです。

### 「型型プログラミング言語」とは何ですか。普通の型付き言語と何が違いますか。

普通の型付き言語では、型は値を分類します。`i32` は整数の集合、`String` は文字列の集合です。

Laplan では、型 (place) が計算の出発点です。型を宣言すると Petri net が生まれ、「この型からあの型へ到達できるか」という問いが成立します。型の型 (metatype) を操作する言語なので「型型」です。値を分類するのではなく、型と型の間の到達可能性を宣言・探索します。

### Petri net を使う必然は何ですか。単なる依存グラフでは足りないのですか。

依存グラフ (DAG) は「A が B に依存する」という静的な関係を表しますが、トークンの消費と生成を扱えません。

API の認証フローでは `refresh_jwt` を使うと消費され、新しい `access_jwt` が生まれます。この「使ったら消える」性質 (`consumes`) は DAG では表現できません。Petri net はトークンの消費・生成・排他性を自然にモデル化でき、solver が到達可能性を探索する際にこれらの制約を正確に反映します。

### Laplan はどんな問題に強く、どんな問題には向きませんか。

**強い問題**: 複数の API を跨いだ認証・データ取得・変換のフロー、多言語 SDK の一括生成、スキーマ変更時の影響診断、client/server 間の型安全な接続です。

**一見向かなさそうで実は扱える問題**: GUI のイベントループは各イベントを 1 step の遷移として分解でき、Petri net 上で自然に表現できます。低レベルのメモリ操作も `axiom/memory/` が load/store の型境界を提供しているため対応できます。

**solver の探索対象にならない問題**: 収束条件が不明な数値計算 (反復収束、未解決問題等) です。これには [unsafe recursive や外部 bridge](architecture/solver.md) で対応できますが、solver は経路として探索しません。

## 入門

### 最初に読むべき文書は README と docs/INDEX.md のどちらですか。

README は「Laplan で何が得られるか」を理解するためのものです。INDEX.md は「何をしたいか」から必要な文書を探すためのものです。初めての方は README を読んだ後、INDEX.md の「やりたいこと別」テーブルから入ってください。

### 既存コードがある人は、invert から入るべきですか。それとも .lex を手で書くべきですか。

状況によります。

- **AT Protocol の Lexicon JSON がある場合**: `.json ↔ .kdl ↔ .lex` の完全可逆変換で JSON からそのまま入力可能
- **Rust / TypeScript 等の既存コードがある場合**: `laplan invert` で `.lex` スケルトンを生成し、`rule` / `chain` を追加
- **ゼロから設計する場合**: `.lex` を手で書き、solver に経路を探索させる [getting-started](guide/getting-started.md) のワークフロー

## 形式と宣言

### .json、.kdl、.lex の 3 形式は、どこまで完全可逆ですか。

`xrpc=` 属性を持つ宣言 (AT Protocol 互換の endpoint 定義) は `.json ↔ .kdl ↔ .lex` が完全可逆です。型名も双方向変換されます (`str` ↔ `string`, `i32` ↔ `integer`)。

`xrpc=` を持たない `.lex` 固有の宣言 (rule, chain, law, derives, family 等) は Lexicon JSON に対応物がないため、`.lex` → `.json` の変換対象になりません。

### .lex を使うと、.json や .kdl では書けない何が書けますか。

rule (到達可能性の宣言)、chain (手動経路固定)、derives (既存宣言からの導出)、law (代数的法則)、family (同型演算の型族)、inverse (逆元関係)、dual (双対クエリ)、consumes (排他的トークン消費) が書けます。詳細は [layers.md](reference/layers.md) を参照してください。

### type、lexicon、rule、func、cratis の違いを、最短でどう理解すればよいですか。

| 概念 | 一言 | Petri net での役割 |
|---|---|---|
| type | 値の種類 | place (トークンが置かれる場所) |
| lexicon | API endpoint の宣言 | 型シグネチャ付きの transition |
| rule | 「何を必要とし何を生むか」 | transition の発火条件 |
| func | 関数本体 | transition の具体的な型制約 |
| cratis | パッケージ | transition のグループ。provides / requires で界面を宣言 |

## IR 層

### Lex₀、Lex₁、Lex₂、Lex₃ は、利用者がどこまで意識すべきですか。

Lex₀〜₃ はコンパイラ内部の分類であり、利用者が直接意識する必要はありません。利用者が意識するのはトップレベルノードの種類です:

| solver が探索する | solver は探索しないが判断材料になる | メタ・外部接続 |
|---|---|---|
| rule, const, assign, inverse, derives, handler/chain, refinement, lex | law, dual, invariant, family | import, cratis, meta |

右列の law / dual / invariant / family は solver の直接の探索対象ではありませんが、個別の型を大量に列挙せずに関係性を書けるため実用上重要です。例えば `derives ... via batch` を使えば getProfile と getProfiles の関係を 1 宣言で表現でき、family を使えば同型演算を持つ型族をまとめて定義できます。

詳細は [layers.md](reference/layers.md) を参照してください。

### solver が探索しないノード (law, dual, family 等) はなぜ重要ですか。

solver は rule / const / assign 等の遷移を探索しますが、law / dual / invariant / family は探索対象ではありません。しかし solver の枝刈り・同一視・導出の根拠として機能します。

例えば family を宣言すれば同型演算を持つ型族をまとめて定義でき、vectorize derives が SIMD transition を自動導出します。law を宣言すれば solver が等価な経路を同一視して探索空間を削減します。dual を宣言すれば forward/reverse の対称性を表現できます。

これらがなくても個別に rule を列挙すれば同じことは書けますが、宣言量が爆発します。構造的な関係を 1 宣言で表現できるのがこれらのノードの役割です。

## Solver の運用

### rule と chain はどう使い分けますか。どの時点で chain が必要になりますか。

rule は「何が必要で何を生むか」を宣言し、solver に経路を任せます。chain は「この順序で実行する」と手動で固定します。

chain が必要になるのは以下の場合です:

- solver が複数経路を返し、制約追加だけでは一意化が困難な場合
- 手順が自明で solver による探索の必要がない場合
- goal を型で表現するのが困難な場合

まず rule で書き、MultiPath 診断が出たら型の精密化を試み、それでも解決しなければ chain を使ってください。

### goal を直接指定する運用と、endpoint を基準に使う運用はどう違いますか。

goal 指定は「profileViewDetailed が欲しい」のように結果を要求する運用です。solver が到達経路を自動発見します。

endpoint 基準は「getProfile を呼ぶ」のように API を直接指定する運用です。chain で順序を固定する場合や、特定の endpoint を必ず経由させたい場合に使います。

goal 指定の方が宣言的で、solver の能力を活かせます。endpoint 基準は既存ワークフローとの互換性が高い運用です。

### 5 段階診断は、実運用でどう読み分けますか。

| 診断 | 読み方 | 次のアクション |
|---|---|---|
| Matched | 成功。経路あり | そのまま使う |
| MultiPath | 複数経路。制約が足りない | rule に requires を追加するか型を精密化する |
| PrunedByBoundary | 認証・権限・データ不足 | capability, ownership, output のどれが遮断しているか確認する |
| PrunedByRefinement | refinement で追加された制約 (認証トークン等) により遮断 | 素の Lexicon では通る経路が capability 追加で遮断された状態。必要な認証情報を marking に追加する |
| MissingFact | 必要な fact がない | rule の produces を追加するか、外部からの input を用意する |

### MultiPath はエラーですか。それとも設計のヒントですか。

設計のヒントです。「同じ距離の複数経路が区別できない」ことを意味し、宣言の制約が足りないことを示しています。具体的な解消手法は [solver と型規律](case/solver-type-discipline.md) を参照してください。

### 最短経路を選ぶ設計だと、本当に正しい経路を選べますか。

最短経路は「最も仮定が少ない解」です。短い経路ほど少ない requires で到達でき、長い経路ほど多くの中間状態を必要とします。経路長が情報量の指標になっています。

正しい経路が最短でない場合、中間に具体的な token を追加すれば正しい最短経路が自然と変わります。あるいは chain で直接固定することもできます。いずれの場合も、曖昧な状態は残りません。

### 想定より短い経路が出たとき、まず疑うべきなのは solver ですか。それとも型宣言ですか。

型宣言です。solver は宣言された世界に忠実であり、宣言外の制約を推測しません。想定より短い経路 (ショートカット) が出た場合、型の精密化が足りず、中間状態が省略されていることがほとんどです。具体例は [solver と型規律](case/solver-type-discipline.md) で扱っています。

### semantic fact と completion token は、いつ導入すべきですか。

**semantic fact**: 共有 fact (`access_jwt` 等) を直接 goal にすると、無関係な endpoint の rule が割り込む場合に導入します。「この endpoint の文脈で生まれた did」のように意味を限定した fact に置き換えます。

**completion token**: endpoint の完走確認が必要な場合に導入します。goal を completion token (`create-account-complete` 等) にすることで、中間の fact だけで到達したとみなされるショートカットを防ぎます。

詳細は [solver と型規律](case/solver-type-discipline.md) を参照してください。

### face、boundary、trust="lexicon-only" は、分散システムでどう効いてきますか。

face は「この視点から何を生成し何を信頼するか」を切り替えます。client face では server を axiom (信頼する前提) として扱い、server face では逆になります。同じ宣言から client 用と server 用の生成物を得られます。

`trust="lexicon-only"` は「相手 face の Lexicon 型宣言だけを信頼し、内部経路は検査しない」モードです。solver は endpoint の出力型を無条件に受け入れ、runtime では出力が Lexicon 型に一致するかを検証します。マイクロサービス間の界面で、相手の内部実装に依存せず型境界だけで接続する設計です。

詳細は [cratis](guide/cratis.md) を参照してください。

## 言語と生成

### resolver.lex は普通の .lex と何が違いますか。なぜ専用の形式が必要ですか。

resolver.lex は runtime solver の実装を FnExpr (Lex₁ の式木) で記述した `.lex` ファイルです。通常の `.lex` は型と rule の宣言ですが、resolver.lex は solver 自体の動作 (経路探索、候補分類、goal 合成) を関数として定義しています。

専用の形式が必要な理由は、solver のロジックを 21 言語全てに synthesis する必要があるためです。Rust で実装すると Rust 専用になりますが、resolver.lex に書けば全 21 言語に自動生成されます。

### 21 言語対応と言っても、各言語でどこまで実用レベルですか。

言語ごとに capability level (L1-L4) で成熟度が異なります。詳細は [target-languages.md](reference/target-languages.md) を参照してください。

- L4 (runtime テスト通過): Rust, Python, Go, Ruby, Kotlin, Java, Lua
- L3 (パッケージビルド通過): TypeScript, Swift, C#, Dart, Zig, Clojure
- L2 (型コンパイル通過): Haskell, C++, D
- L1 (構文 lint 通過): JavaScript, PHP, OCaml, Elixir, Gleam

### 新言語追加で本当に Rust 側の実装変更が不要なのは、どの範囲までですか。

mapping.lex (型マッピング)、morph.lex (構文パターン)、type.lex (型変換) の 3 ファイルだけで完結するのは、言語の構文がブレース系 (`{ }` で block を閉じる) か end キーワード系 (`def...end`) の場合です。

Rust 側の変更が必要になるのは、(1) inverse の `free_fn.rs` に新しい関数抽出パターンが必要な場合 (Python のインデントベース構文など)、(2) 既存の共通パーサでカバーできない構文構造がある場合です。詳細は [adding-language.md](guide/adding-language.md) を参照してください。

### bindings {}、package {}、runtime-base {} の 3 層は、なぜ分かれているのですか。

3 層はそれぞれ異なる責務を持ちます。

| 層 | 責務 | 例 |
|---|---|---|
| `bindings {}` | axiom の射を外部関数に接続 | `"signature.verify.ES256K"` → `k256::ecdsa::VerifyingKey::verify` |
| `package {}` | 生成パッケージの manifest テンプレート | `[dependencies] k256 = "0.13"` |
| `runtime-base {}` | 生成コードの共通型と helper | `UnknownValue`, `Bytes`, `BlobRef` の型定義 |

`bindings` で指名した関数がコンパイル時に解決されるよう、`package` が manifest に dependency を追加し、`runtime-base` が共通型を提供します。分離することで、同じ axiom に対して言語ごとに異なる接続先を差し替えられます。詳細は [axiom-bindings.md](guide/axiom-bindings.md) を参照してください。

### runtime solver だけ MPL-2.0 になる理由は何ですか。

通常の生成物 (型宣言、handler 骨格、decoder) は利用者の `.lex` 宣言と mapping.lex テンプレートから決定論的に導かれるため、MIT です。利用者のデータの派生物であり、laplan 固有のアルゴリズムを含みません。

runtime solver の生成物は laplan 本体の探索アルゴリズム、経路検証、dispatch 戦略を含むため、MPL-2.0 です。MPL-2.0 はファイル単位の copyleft であり、改変した solver ファイルの公開義務はありますが、周囲の MIT 出力や利用者のコードには伝播しません。詳細は [generated-output-licenses.md](reference/generated-output-licenses.md) を参照してください。
