# Solver と型規律

solver は rule の型宣言から Petri net の到達可能性を探索します。探索結果の正しさは、型宣言がドメインの制約を正確に反映しているかに依存します。宣言された世界に忠実であり、宣言外の制約は推測対象外です。

このケーススタディでは、solver が返すショートカット (期待より短い経路) を型の精密化で解消する手法を扱います。

## 暗黙の依存と構造的保証

手続きプログラムの実行順序は暗黙の依存関係を作ります。以下は「アカウントを作成し、セッションを発行する」手続きの典型:

```rust
fn create_account(&mut self, input: CreateAccountInput) -> Result<CreateAccountOutput, ...> {
    let did = self.generate_did(...)?;
    self.store_account(...)?;              // ← ここが失敗したら?
    let tokens = self.issue_tokens(did)?;  // ← store 失敗でも token 発行の余地
    Ok(CreateAccountOutput { did, tokens })
}
```

`issue_tokens` は `store_account` の成功に依存しますが、その依存は 2 つの暗黙に支えられています:

1. `store_account` が `issue_tokens` より先に実行される (行順序)
2. `store_account` の失敗がエラー伝搬される (`?` 演算子)

いずれもプログラマの注意力に依存します。`store_account` が例外を投げずに黙って失敗するライブラリだったら、`issue_tokens` はアカウントが存在しないまま token を発行します。

rule の型では同じ制約が構造的に表現されます:

| 手続き的保証 | 型による保証 |
|---|---|
| 行順序で store が先 | `issue_tokens` が `account` を requires |
| `?` でエラー伝搬 | store 失敗時に `account` が marking に不在、`issue_tokens` は発火不能 |

**実行順序依存は型の不在が作ります。** 型として宣言されていない中間状態は solver の探索空間に存在しません。手続きコードでは「その行を通過した」が成功の証拠ですが、rule では「その型が marking に存在する」ことだけが証拠になります。

## ショートカットの解釈

solver が期待より短い経路を返した場合、それは型の不備を示します。

| 症状 | 原因 | 修正方針 |
|---|---|---|
| 読み取りと書き込みが同じ型を返す | 書き込み完了の事実が型として存在しない | 格納証拠型を追加 |
| 異なる操作が同じ requires で発火する | input type の粒度が粗い | 操作固有の前提条件を requires に追加 |
| 認証なしで保護リソースに到達する | capability が未宣言 | capability を rule に追加 |
| 単一 step で goal に到達する | 中間状態が型として存在しない | 中間生成物を produces/requires で連鎖させる |
| 別 endpoint の経路が goal に割り込む | 共有 fact を直接 goal にしている | semantic fact / completion token に置換 |

**ショートカットは solver のバグではなく、宣言の不足に対する診断です。**

## 収束の段階

solver の経路が 1 つに収束するまで、型を精密化する反復:

| 経路数 | 意味 | 次の行動 |
|---|---|---|
| 0 | 到達不能。producer が不在 | missing facts を確認し、rule を追加 |
| 1 | 収束。型がドメイン制約を十分に反映 | 完了 |
| 2+ | 分岐。同じ goal に到達する経路が複数 | 下表で原因を分類 |

分岐の原因分類:

| 原因 | 例 | 修正 |
|---|---|---|
| 経路長の差 | handle → resolve → profile (2-step) と did → profile (1-step) が共存 | solver は短い方を選ぶ。長い方しか正しくない場合、短い方の requires が不足 |
| 粒度不足 | 読み取りと書き込みが同じ型を返す | 書き込み証拠型を分離 |
| 前提条件の欠落 | 認証なしで保護リソースに到達 | capability を追加 |
| 共有 fact の汚染 | 別 endpoint が同じ `access-jwt` を produces して割り込む | semantic fact に置換 |

solver は最短経路を優先します。複数経路が異なる深さで共存する場合、短い方が「型の制約不足で成立している不正なショートカット」である可能性が高いです。

## 格納証拠型

### 読み取りと書き込みが同一型を返す場合

DB にアカウントを格納する操作 (`store_account`) と、DB からアカウントを読み取る操作 (`get_account`) がともに `account` 型を `produces` に持つ場合、solver は読み取り経由のショートカットを発見します。

「新規作成」の文脈では **DB に格納されたという事実** が必要であり、既存レコードの読み取りでは代替できません。

修正として、格納操作に固有の証拠型を追加します:

```kdl
rule "store_account" {
    requires output="did"
    produces output="account"
    produces output="account-record"    // 格納証拠: store からのみ得られる
}
```

goal に `account-record` を含めれば、solver は `store_account` を経由する経路のみを解とします。`get_account` は `account` を `produces` に持ちますが `account-record` は持たないため、ショートカットは排除されます。

## Semantic fact と completion token

### 共有 fact への直接依存

`access-jwt`, `refresh-jwt`, `did`, `session-output` のような共有 fact をそのまま goal や requires に使うと、別のエンドポイントや rule が最短経路に割り込みやすくなります。

例えば、`createSession` と `refreshSession` がともに `access-jwt` を `produces` に持つ場合、`access-jwt` を goal にしたエンドポイントの検証では両方の経路が出現し、意図した経路を特定できません。

### エンドポイント固有の fact による経路の限定

共有 fact を直接使わず、2 種類の endpoint 固有 fact を導入します。

**Semantic fact**: 操作の意味的な完了を表す fact。共有 fact を endpoint 固有の文脈に変換します。

```kdl
// 共有 fact (did) をそのまま使うと他の endpoint が割り込む
rule "generate_did" {
    requires output="handle"
    produces output="did"           // 共有 fact
    produces output="created-did"   // semantic fact: この endpoint の文脈
}
```

**Completion token**: endpoint 全体の完走を示す fact。endpoint の goal として使います。

```kdl
rule "issue_session_pair_from_account_record" {
    requires output="account-record"
    produces output="access-jwt"
    produces output="refresh-jwt"
    produces output="create-account-complete"   // completion token
}
```

### 運用パターン

1. raw input / 共有 fact → request rule でエンドポイント固有の token に変換
2. write / mutation rule の requires を semantic fact のみに限定
3. エンドポイントの完走確認は completion token を goal に指定

```
// createAccount の経路
validate_handle      : input → validated-handle
check_handle_unique  : validated-handle → handle-unique
generate_did         : handle-unique → created-did, account-create-ready
store_account        : created-did, account-create-ready → account-record
issue_session_pair   : account-record → create-account-complete
```

各 rule の requires/produces が endpoint 固有の fact で繋がるため、他の endpoint の rule が経路に割り込めません。solver は `create-account-complete` を goal にした探索で、この 5-step の経路 1 本に収束します。

## 外部依存の信頼境界

格納証拠や semantic fact で solver が検証するのは category 内部の型連鎖であり、category の外側 (axiom) は信頼の対象ですが検証の対象外です。

| 範囲 | solver の関与 | 例 |
|---|---|---|
| category | 経路の全列挙、到達可能性の証明 | rule 間の型連鎖、中間状態の網羅 |
| axiom | 信頼する。検証しない | import したライブラリの内部実装、外部 API のレスポンス、DB の書き込み保証 |

category 内の rule が axiom の操作を経由することを `composed-of` で宣言できます:

```kdl
rule "store_account" {
    requires output="created-did"
    requires output="account-create-ready"
    composed-of "datum.record.create"    // axiom 経由を明示
    produces output="account"
    produces output="account-record"     // 格納証拠
}
```

`composed-of` は「この rule は外部操作を経由する」ことを宣言します。`datum.record.create` が RDB への INSERT なのか KV store への PUT なのかは solver の関知外です。solver が保証するのは「`account-record` を得るには `datum.record.create` を経由する rule を通る以外にない」という category 内の構造です。

### 信頼レベルと型の設計

外部システムとの接点では、レスポンスの信頼度が型の設計に影響します。

| 信頼レベル | 例 | 型の扱い |
|---|---|---|
| 完全信頼 | 自プロセス内の純粋な計算 | produces で直接生成 |
| 条件付き信頼 | import したライブラリの戻り値 | ライブラリの型境界を axiom として宣言し、戻り値型を category に取り込む |
| 要検証 | 外部 API のレスポンス | 検証 rule を requires に挟む。レスポンスをそのまま信頼しない |

外部 API のレスポンスを直接 `produces` として扱うと、API が不正な値を返した場合にも後続の rule が発火します。検証 rule を挟むことで、「検証済みレスポンス」という別の型が category 内に生まれ、未検証のレスポンスでは後続に進めなくなります。

## solver 出力の解釈指針

| solver の報告 | 解釈 | 行動 |
|---|---|---|
| 最短経路が期待より短い | 型の制約が不足。中間状態が暗黙 | 不足する中間型を追加 |
| 複数経路が異なる深さで共存 | 短い経路が型の不備で成立している可能性 | 短い経路の各 step を検査し、本来必要な requires が欠けていないか確認 |
| 別 endpoint の rule が割り込む | 共有 fact を goal にしている | completion token を goal にし、共有 fact への直接依存を semantic fact に置換 |
| 到達不能 | producer が不在、または requires が過剰 | missing facts で不足を特定 |
