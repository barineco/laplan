# Semantic Fact Design: fact 命名と rule 構造の設計コンベンション

solver が経路探索する際に「混線」(意図しない shortcut が紛れ込む / 候補が爆発する) を起こさず、shortest route が一意に確定するための fact / rule 設計のコンベンションです。

> 関連: [layers.md](layers.md), [case/solver-type-discipline.md](../case/solver-type-discipline.md)

## 1. 経験則 (rules)

### R1. raw 共有 fact を直接 `requires` に書かない (混線源)

`handle` / `did` / `access-jwt` のような汎用名の fact を transition の `requires` に直接書くと、別経路の rule が同名 fact を `produces` に持っていた場合に混線します。

raw 入力は `assign` で原 fact として供給し、各 rule は **origin 付きの semantic fact** を介して受け渡します。

### R2. `produces` は 1 rule = 1 semantic fact が原則 (hub 化防止)

1 rule が複数 fact を `produces` に持つと、それらを使う他経路すべてがその rule を経由可能になり中継 hub の形になり、候補が爆発します。副次的に得られる fact は別 rule に分離します。

```
× rule "validate_access_session" {
×     requires output="access-jwt"
×     produces output="did"          // ← 他経路と混線
×     produces output="account"
×     produces output="validated-session-view"
×     produces output="validated-access-session"
× }

○ rule "validate_access_session" {
○     requires output="access-jwt"
○     produces output="validated-access-session"
○ }
○ rule "session_context_from_access_session" {
○     requires output="validated-access-session"
○     produces output="did.from-access-session"
○     produces output="account.from-access-session"
○     produces output="validated-session-view.from-access-session"
○ }
```

### R3. fact 名は origin と意味を一意に表す semantic tag (後置統一)

命名規約は **`<variable>.<semantic-tag>`** の後置形式で一貫させます。

利点:

- 同一 variable の全用途が alphabetical sort で隣接します (`handle.create-account`, `handle.resolve-handle` が並びます)
- tag 位置が末尾固定なので grep / pattern matching の規則性が保てます
- variable 名と semantic tag が混同されません

### R4. `assign` / `const` は initial marking 提供であり transition には数えない

`assign` / `const` は `() → T` の initial-marking 提供であり、探索 step に含めません。compile crate の axiom_facts として face ごとに集約し、joint_search 起動時に initial marking に合一します。

これにより rule chain の論理 depth (rule 通過数) と探索 depth が一致します。`assign` が複数あっても depth は膨らみません。

### R5. shortcut 発見は型不備の指摘

solver が予期せぬ shortcut (例: 「validate を経由していない did で session を発行できる」) を返したら、それは fact 設計の不備です。型を精密化する機会として捉えます。具体的な対応は以下です。

- raw 共有 fact を semantic 化する (R1)
- 通過 token を導入する (R6)
- `produces` を絞る (R2)

詳細な事例は [case/solver-type-discipline.md](../case/solver-type-discipline.md) を参照してください。

### R6. 通過 token (rule 通過証拠 fact) で chain を確定する

後続 rule の前提として「特定 chain を通った」ことを表す fact を `produces` に持たせます。これにより raw input から完了までの chain が一意になります。

例:

- `validated-handle` (validate_handle 通過の証拠)
- `unique-handle` (check_handle_uniqueness 通過の証拠)
- `account-create-request` (rule 名と一致、それ自体が形成証拠)
- `account.created` (entity.event 形)
- `<endpoint>-complete` (endpoint 完了 token)

### R7. ownership など別系統の concept も semantic 化で表現する

`requires ownership="did"` のように Fact::SelfKey 系で曖昧に表現するより、`requires output="did.from-access-session"` と semantic 化したほうが意味が明確になり混線も防げます。

admin 操作なら `did.is-admin-confirmed` のような別 origin tag を作って表現します。「自分の did を持っている」を別 transition の通過証拠として明示する形です。

## 2. 命名規約まとめ

| 種別 | 命名 | 例 |
|---|---|---|
| input fact (`assign` 由来) | `<variable>.<endpoint-context>` | `handle.create-account`, `password.create-session` |
| 通過 token (派生形) | `<verb>-<variable>` | `validated-handle`, `unique-handle` |
| 通過 token (rule 名一致) | `<rule-name>` | `account-create-request`, `login-request` |
| 状態変化 token | `<entity>.<event>` | `account.created`, `did.generated` |
| 文脈付き派生 | `<entity>.from-<source>` | `did.from-access-session`, `account.from-credentials` |
| 完了 token | `<endpoint>-complete` | `create-account-complete` |
| family / subtype (Fact poset) | family の members で表現 | `did` ∈ `at-identifier` family |

semantic tag は **末尾固定** (`.from-X`, `.<event>`, `<endpoint-context>`) を貫くことで grep / sort の規則性を確保します。

## 3. 設計プロセス (修正前のチェック)

1. **全体 read** で fact 依存マップを把握します。grep ピンポイント修正は相互作用を見落とすので避けます
2. 共有 fact が **origin で分離可能か** を確認します (R1)
3. `produces` が **1 rule = 1 semantic fact** に絞れるかを確認します (R2)
4. `assign` / `const` が **transition 化されていないか** を確認します (R4)
5. 修正後に **全 endpoint の経路と経路の中身** をレビューします (depth だけでなく route 内容を確認します)

## 4. アンチパターン

| パターン | 症状 |
|---|---|
| grep ピンポイント修正 | fact 同士の相互作用を見落とし、整合性が破壊されます |
| validate 系 rule が副次 fact を `produces` に持つ | 中継 hub の形になり候補が爆発します |
| `assign` を transition 化する | depth が膨張し、permutation で爆発します |
| raw fact (handle / did) を直接 `requires` に書く | 別経路と混線します |
| ownership など別系統 fact を semantic 化せず残す | 「自分の」の意味が複数 origin と曖昧になり混線源になります |

## 5. witness との関係

`Fact::Witness { endpoint }` を first-class 化した場合、本文書 R6 の **完了 token (`*-complete`) 部分は手書きが不要** になります。endpoint 宣言から自動生成される witness が同等の役割を果たし、shortcut も endpoint nsid の一意性で構造的に不可能になります。

ただし R1-R5, R7 (raw input の semantic 化、`produces` 単一化、origin tag、initial marking 化、ownership 表現) は witness とは独立のレイヤーで、依然手で設計する必要があります。witness は完了側の話、本文書は入力 / 中間 fact 側の話です。

移行イメージ:

- 現状: `produces output="create-account-complete"` を手書き (R6 の通過 token として)
- witness 自動生成後: 該当 rule は `Fact::Witness { endpoint: "..." }` を自動で `produces` に持ち、手書き token は削除可能になります

## 6. 適用イメージ

PDS の createAccount endpoint を考えます。`assign` で `handle` / `password` / `email` を semantic 化します。

```
assign "handle.create-account" type="String" from="$input.handle"
assign "password.create-account" type="String" from="$input.password"
assign "email.create-account" type="String" from="$input.email"
```

rule 群はこれらを semantic fact として `requires` に取り、各 step で通過 token を `produces` に持たせます。

```
rule "validate_handle" {
    requires output="handle.create-account"
    produces output="validated-handle"
}
rule "check_handle_uniqueness" {
    requires output="validated-handle"
    produces output="unique-handle"
}
rule "generate_did" {
    requires output="unique-handle"
    requires capability="signing-key"
    produces output="did.generated"
    produces output="created-did"
    produces output="account-create-ready"
}
rule "store_account" {
    requires output="created-did"
    requires output="account-create-ready"
    requires output="unique-handle"
    requires output="password-hash"
    requires output="email.create-account"
    requires output="account-create-request"
    produces output="account.created"
}
rule "issue_session_pair_from_created_account" {
    requires output="account.created"
    produces output="create-account-complete"
}
```

これにより solver は createAccount 経路を一意に確定でき、resolve_handle 等の別経路から `did` を持ってきた shortcut は構造的に不可能になります (`did.generated` と `did.from-resolved-handle` は別 fact です)。

## 関連 docs

- [layers.md](layers.md): .lex トップノードと層分類
- [case/solver-type-discipline.md](../case/solver-type-discipline.md): 型精密化が安全性を構造化する事例
- [architecture/solver.md](../architecture/solver.md): solver の探索アルゴリズムと枝刈り
