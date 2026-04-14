# Laplan: Lexicon as Programming Language

[English](README.md)

> **注意:** この文書には予定・構想段階の内容を含みます。リリース時に変更される可能性があります。

KDL 形式で型と関係を宣言し、Petri net 経路解決で合成する型型プログラミング言語。

## 型型プログラミング言語

Lexicon は API endpoint の入出力を型で宣言する。

Laplan はこの「型を定義する」行為そのものを言語にした、型の型 (metatype) の言語。

`did` と `profile` という 2 つの型があれば「did から profile への経路は?」という問いが成立し、solver が経路を自動探索する。ただし型だけでは制約が足りない。JWT トークン、有効期限、リソース所有権など API 定義の外にある暗黙の前提条件を morphism の制約として注釈することで、solver は有効な経路だけを発見する。

この経路探索は Petri net の到達可能性問題として定式化される。solver の設計は Lean の tactic に着想を得ており、抽象度の高い制約 (law, morphism) ほど solver が強く働き、具体的な手順に降りるほど手動で経路を示す構造は、`ring` / `simp` と手動 `apply` の関係に対応する。経路が到達証明であること、制約で到達性が変わること、注釈が型から復元不能であることは Lean 4 / Mathlib で形式検証済み ([lean-lexicon](https://github.com/barineco/lean-lexicon))。関連: [lean-eigenraum](https://github.com/barineco/lean-eigenraum) (振動系エネルギー輸送の形式検証、136 定理, zero sorry)。

## 何が可能になるか

- client/server を跨いだ型安全
  認証トークンの検証を型の制約として宣言し、API を叩く前にエラーを検知
  
- 暗黙の前提条件の明示化
  HTTP ヘッダの JWT やトークン期限切れなど、コードに散らばっていた暗黙知を宣言に集約
  
- フレームワーク非依存
  axum, Express 等の固有型に縛られない統一的な型宣言
  
- SIMD / GPU / threading の自動最適化
  経路探索の副産物として最適な命令列を発見
  
- 22 言語への変換 + 逆変換
  既存ワークフローに接続し、一部だけ借りる運用が可能
  
- 全経路の列挙と診断
  探索空間が有限なため、全ての経路を網羅し、不足情報を報告できる

---

## 3 つの形式

同じ API 定義 (`app.bsky.actor.getProfile`) を 3 つの形式で比較する:

### `.json` (Lexicon JSON: AT Proto 公式原本)

```json
{
  "lexicon": 1,
  "id": "app.bsky.actor.getProfile",
  "defs": {
    "main": {
      "type": "query",
      "parameters": {
        "type": "params",
        "required": ["actor"],
        "properties": {
          "actor": {
            "type": "string",
            "format": "at-identifier"
          }
        }
      },
      "output": {
        "encoding": "application/json",
        "schema": { "type": "ref", "ref": "app.bsky.actor.defs#profileViewDetailed" }
      }
    }
  }
}
```

### `.kdl` (Lexicon KDL: JSON の忠実な人間可読射影)

```kdl
lexicon app.bsky.actor.getProfile version=1 {
    query {
        params {
            (string)actor required=#true format=at-identifier
        }
        output "app.bsky.actor.defs#profileViewDetailed" encoding="application/json"
    }
}
```

### `.lex` (Laplan KDL: 計算言語)

```kdl
app.bsky.actor.getProfile xrpc=query {
    in { (str)actor format=at-identifier }
    out { (ref)profile type="app.bsky.actor.defs#profileViewDetailed" }
}

morphism "app.bsky.actor.getProfile" {
    requires input="actor"
    produces output="profile"
}
```

- Laplan では `required` がデフォルト、`?` で optional を明示する

- `query` / `procedure` は `xrpc=` annotation で保持し、入出力は `in` / `out` に統一

- 型名は Laplan 正式形 (`str`, `i32`, `f64` 等)。morphism を宣言すれば solver が経路を解く。

`xrpc=` の有無が 2 つの世界を排他的に分ける:

| | `xrpc=query\|procedure\|subscription` | `xrpc` なし |
|---|---|---|
| 名前空間 | NSID (reversed domain) | Laplan namespace (自由形式) |
| required デフォルト | optional (`?` 不要、Lexicon 互換) | **required** (`?` で optional) |
| round-trip | `.json ↔ .kdl ↔ .lex` 完全可逆 | Lexicon JSON に対応物なし |
| 例 | `app.bsky.actor.getProfile` | `math.arith.add`, `io.std.print` |

---

### Laplan でのみ表現可能なもの

Lexicon JSON は 1 ファイル = 1 NSID であり、各 endpoint が孤立している。KDL 形式 (`.kdl` / `.lex`) では**1 ファイルに複数の lexicon を記述**でき、関連する宣言をまとめて管理できる。以下は Lexicon では記述できず、Laplan で初めて表現可能になる:

| 機能 | 説明 | キーワード |
|---|---|---|
| 非 API 関数 | HTTP に縛られない算術・論理等 | `in` / `out` |
| 副作用宣言 | 何が必要で何を生むか | `morphism` |
| 導出 | 既存宣言のバッチ化・合成・変換 | `derives` |
| 合成パイプライン | 複数ステップの handler チェーン | `handler` / `chain` |
| 逆元関係 | action/undo のペア | `inverse` |
| 双対クエリ | forward/reverse の可換条件 | `coherence` |
| 代数的法則 | 群公理、冪等性、準同型 | `law` |
| リソース消費 | 排他的トークン消費 (一般ペトリネット) | `consumes` |

```kdl
// 算術関数 (API ではない)
math.arith.add {
    in { (i32)a; (i32)b }
    out { (i32)result }
}

// 副作用の宣言 (何が必要で何を生むか)
morphism "app.bsky.actor.getProfile" {
    requires input="actor"
    produces output="profile"
}

// 既存宣言からの導出 (getProfile → getProfiles をバッチ化)
derives "app.bsky.actor.getProfiles" from "app.bsky.actor.getProfile" via batch {
    singular-input "actor"   plural-input "actors"
    singular-output "profile" plural-output "profiles"
}

// 関数の合成パイプライン
handler "ink.illo.dm.send" {
    chain {
        step "server.identity.resolve_pubkey" input="recipient-did" output="recipient-pubkey"
        step "crypto.vault.create_sealed_dm" input="content,recipient-pubkey" output="sealed"
        step "server.blockstore.write_record" input="sealed" output="uri,cid"
    }
}

// 逆元関係
inverse mute_actor {
    action "app.bsky.graph.muteActor"
    inverse "app.bsky.graph.unmuteActor"
    kind reversible
}

// 双対クエリ (forward/reverse の可換条件)
coherence follow_dual {
    record "app.bsky.graph.follow"
    forward "app.bsky.graph.getFollows" direction=outgoing key="subject"
    reverse "app.bsky.graph.getFollowers" direction=incoming key="subject"
}
```

Lexicon は「この API は何を受け取り何を返すか」を定義する。Laplan は「この API は他の何と関係し、どう合成され、何を前提とするか」を定義する。

---

## 概要

Laplan は AT Protocol の [Lexicon](https://atproto.com/specs/lexicon) スキーマ形式に着想を得た言語。Lexicon が API エンドポイントを宣言するように、Laplan は**計算の型付きインターフェース**を宣言する。

各宣言は Petri net 上の transition となり、`requires`（必要なトークン）と `produces`（生成するトークン）を満たす経路を solver が自動的に発見する。プログラマは「何を持っていて、何が欲しいか」を書き、Laplan が経路を解く。

### Lexicon との関係

| | Lexicon | Laplan |
|---|---|---|
| 形式 | JSON (`.json`) / KDL (`.kdl`) | KDL (`.lex`) |
| 目的 | API スキーマ定義 | 計算の型付き記述・合成 |
| 名前空間 | NSID (reversed domain) | 自由 (ドット区切り) |
| 合成 | なし (呼ぶだけ) | coherence / derives / Petri net |
| required | optional がデフォルト | **required がデフォルト** |
| 型名 | `string`, `integer`, `number` | `str`, `i32`, `i64`, `f64` |

3 形式は相互変換可能:

| 変換 | 可逆性 |
|---|---|
| `.json` <-> `.kdl` <-> `.lex` (`xrpc=` 付き) | 完全可逆 (型名も双方向変換) |
| `.lex` (`xrpc=` なし) | Laplan ネイティブ; Lexicon に対応物なし |

---

## 宣言の書き方

### 1. 型 (Place): 全ての根源

Laplan の出発点は型の定義。Petri net の **place** に対応する。

| Laplan | 説明 | Lexicon | WASM |
|---|---|---|---|
| `i32` | 符号付き 32-bit 整数 | `integer` | i32 |
| `i64` | 符号付き 64-bit 整数 | (なし) | i64 |
| `f32` | 32-bit 浮動小数点 | (なし) | f32 |
| `f64` | 64-bit 浮動小数点 | `number` | f64 |
| `bool` | 真偽値 (0/1) | `boolean` | i32 |
| `str` | UTF-8 文字列 | `string` | host string |
| `bytes` | バイト列 | `bytes` | host bytes |
| `any` | 任意の構造化値 | `unknown` | host value |

構造化された型:

```kdl
(object)profileView {
    (str)did format=did                // 括弧で型名 + フィールド名
    (str)handle format=handle
    (str)displayName? max-length=640   // ? で optional
    (data.map)metadata?                // 括弧内にパスでカスタム型
}
```

### 2. Morphism (Transition の宣言): 型の間の関係

型と型の間に**到達可能性**を宣言する。Petri net の transition。morphism は「何が必要で、何が生まれるか」だけを宣言し、実装は bridge に委譲される。

```kdl
// ログイン: 認証トークンを生成する
morphism "com.atproto.server.createSession" {
    produces capability="access_jwt"
    produces capability="refresh_jwt" ttl=86400
}

// トークン更新: refresh を消費して新しい access を生成する
morphism "com.atproto.server.refreshSession" {
    requires capability="refresh_jwt"
    consumes capability="refresh_jwt"      // 一度使ったら消える
    produces capability="access_jwt" ttl=3600
}

// プロフィール取得: access_jwt が必要
morphism "app.bsky.actor.getProfile" {
    requires capability="access_jwt"
}
```

認証トークンの有無、有効期限 (`ttl`)、排他的消費 (`consumes`) が型レベルの制約になる。solver は `createSession → getProfile` の経路を自動発見する。`access_jwt` が期限切れの場合、solver は「何が足りないか」から逆算して再認証経路 (`refreshSession`) を自動的に挿入する。if 文によるトークン検証のハードコードは不要になり、宣言された制約から必要な手順が導出される。

### 3. 関数 (Transition の制約): 型シグネチャの具体化

morphism だけでは型の関係があるだけ。関数はその経路に**具体的な型シグネチャ**を与える。

```kdl
app.bsky.actor.getProfile xrpc=query {
    in { (str)actor format=at-identifier }
    out { (ref)profile type="app.bsky.actor.defs#profileViewDetailed" }
}

data.json.parse {
    in { (str)input }
    out { (any)value }
    errors {
        error InvalidJson description="不正な JSON"
    }
}
```

関数は transition を「これが入り、これが出る」と制約する。制約が十分なら、solver の経路は自然と 1 つに収束する。

### 4. Chain: 手動経路固定

solver の自動経路ではなく、step の順序を明示する。

```kdl
handler "ink.illo.dm.send" {
    chain {
        step "server.identity.resolve_pubkey" input="recipient-did" output="recipient-pubkey"
        step "crypto.vault.create_sealed_dm" input="content,recipient-pubkey" output="sealed"
        step "server.blockstore.write_record" input="sealed" output="uri,cid"
    }
}
```

### 型 → Morphism → 関数 → Chain: 制約の積み重ね

| 層 | 宣言する内容 | スタイル |
|---|---|---|
| **型** (place) | 何があるか | 最も宣言的 |
| **Morphism** | 何から何へ行けるか (方向) | |
| **関数** | 何が入り何が出るか (型シグネチャ) | |
| **Chain** | どの順で通るか (手順) | 最も命令的 |

> 上ほど solver に任せる。下ほどプログラマが指定する。

**上だけで十分なら下は省略できる。** morphism だけで solver が一意の経路を見つければ、関数も chain も不要。足りなければ下の層で制約を追加する。

高レイヤー (API) では morphism だけで solver に任せるのが自然。「DID を持っている、プロフィールが欲しい」で十分。低レイヤー (算術) でも、代数的法則 (`law`) を宣言すれば solver が等価な経路を同一視し、型の依存関係から実行順序と並列性が導出される。chain は理論上必須ではないが、goal を型で宣言するのが困難な場合や、構造制約を書き下すより手順を直接示す方が明快な場合に有用。

頻繁に使う solver 経路は chain に固定できる。

後述のビルドオプションでも固定可能で、この場合 solver の経路発見コストがゼロになり、確定的な関数が得られる。

### Solver の動作

solver は Petri net の到達可能性を**幅優先探索 (BFS)** で解く。

#### 3 つの概念

| 概念 | 定義 | 例 |
|---|---|---|
| **Marking** | 現在持っているトークンの集合 (place 上のトークン) | `{ Capability("access_jwt"), Output("did"), SelfKey("self.repo") }` |
| **Goal** | 到達したいトークン | `[ Output("profile") ]` |
| **Transition** | requires を確認し、consumes トークンを消費し、produces を生成する遷移 | 下記参照 |

| Transition | requires | produces |
|---|---|---|
| `resolve_handle` | _(なし)_ | `Output("did")` |
| `getProfile` | `Output("did")` | `Output("profile")` |
| `getFollowers` | `Output("did")` | `Output("followers")` |

#### BFS の流れ

> **初期 marking**: `{ Input("handle") }` | **Goal**: `[ Output("profile") ]`

0. **深さ 0** ... marking: `{ Input("handle") }`
   - 発火可能: `resolve_handle` (requires=[] → 常に発火可能)
1. **深さ 1** ... marking: `{ Input("handle"), Output("did") }`
   - 発火可能: `getProfile` (requires=[did] ✓)
   - 発火可能: `getFollowers` (requires=[did] ✓) ... 両方発火可能だが...
2. **深さ 2**
   - `getProfile` 経由 → `{ ..., Output("profile") }` ... **goal 到達!**
   - `getFollowers` 経由 → `{ ..., Output("followers") }` ... goal 未到達

> **結果**: 1 経路 `[ resolve_handle → getProfile ]`

solver は全ての発火可能な transition を幅優先で試し、goal に最短で到達する経路を返す。produces が goal と一致する transition だけが経路に残る。

#### 複数 biblio の結合

複数の biblio を同時に読み込むと、全ての transition が一つの TransitionTable に合流する。biblio 間の API 界面が接触するところで、solver が biblio を跨いだ経路を発見する。

| biblio | Transition | requires | produces |
|---|---|---|---|
| `atproto/` | `resolve_handle` | _(なし)_ | `did` |
| `bsky/` | `getProfile` | `did` | `profile` |
| `bsky-cli/` | `display_profile` | `profile` | _(表示)_ |

> **Solver 結果**: `resolve_handle` → `getProfile` → `display_profile` (biblio 横断経路)

各 biblio は自分の provides / requires だけを宣言する。結合は solver が行う。

#### 経路の収束と分岐

**produces が異なれば経路は自然と 1 つに定まる。** 上の例で `getProfile` は `profile` を、`getFollowers` は `followers` を produces する。goal が `profile` なら `getProfile` しか選べない。

分岐が生じるのは **同じ goal を produces する transition が複数あるとき**:

| Transition | requires | produces |
|---|---|---|
| `getProfileByDid` | `did` | `profile` |
| `getProfileByHandle` | `handle` | `profile` |

> **marking**: `{ did, handle }` | **goal**: `[ profile ]` → solver: **2 経路** (どちらも profile を produces する)

これは「DID からもハンドルからもプロフィールが取れる」という正当な分岐。解消方法:

1. **marking を絞る**: `{ did }` だけなら `getProfileByDid` の 1 経路
2. **requires を追加**: 一方に capability を付けて差別化する
3. **chain で固定**: 使いたい経路を明示する

実際の AT Protocol で 3 経路が出た事例: Lexicon JSON をそのまま解いたとき、HTTP ヘッダーの JWT トークンが API 定義の外にあったため、`getProfile` / `getBlocks` / `getFollows` の requires がすべて `[did]` だけになった。capability を morphism に明示したら 1 経路に収束した。

**複数経路は設計フィードバック。** solver が分岐を返したら、requires / produces の宣言を見直す合図。

#### 到達不能と不足情報の診断

goal に到達できない場合、solver は何が足りないかを分類して報告する:

| 診断 | 意味 | 例 |
|---|---|---|
| **AlreadySatisfied** | goal は既に marking にある | |
| **Reachable** | 経路が見つかった | `[resolve_handle → getProfile]` |
| **Recoverable** | 一部の fact を補えば到達可能 | 「did があれば行ける」 |
| **NeedsUserAction** | ユーザの入力が必要 | 「actor の DID を入力してください」 |
| **PrunedByBoundary** | 境界条件で遮断された | 認証・権限・データ不足 |

`BoundaryKind` は遮断の理由を 3 種に分ける:
- **Capability**: 認証トークンがない (例: `access_jwt`)
- **Ownership**: 自分のリソースでない (例: 他人の repo に書き込もうとした)
- **Output**: 必要なデータがない (例: DID が未解決)

この診断により、「何を持っていて、何が足りなくて、何があれば到達できるか」がプログラマに提示される。

#### 二層 solver

solver は二層構成で動作する。同一の BFS アルゴリズムが Fact の種別を変えて再利用される。

| 層 | 対象 | 入力 | 出力 | Fact の例 |
|---|---|---|---|---|
| Layer 1: morphism solver | API 経路 | marking (capability 等) + goal | Recipe (NSID 列) | `Capability("access_jwt")`, `Datum("profile")` |
| Layer 2: instruction solver | WASM 命令選択 | 値レベル marking + goal | Recipe (opcode 列) | `Value { ty: "f64", name: "a" }`, `SimdValue { ty: "f64x2", name: "ab" }` |

Layer 2 は kind の `vectorize` derives から生成された SIMD transition をスカラー transition と同一テーブルに配置する。BFS が最短経路を発見するとき、SIMD 経路がスカラーより短ければ自動的に選択される。

```
// 例: sum_of_squares
// スカラー経路 (深さ 3): f64.mul → f64.mul → f64.add
// SIMD 経路   (深さ 2): f64x2.mul → f64.add
// → BFS は深さ 2 の SIMD 経路を先に発見する
```

**SIMD 最適化は最適化パスではなく、Petri net 上の最短経路発見の帰結。**

### 5. Kind: 同型演算の型束

型の間の同型関係を宣言する。kind は「これらの型は同じシグネチャの演算を持つ」ことを表す。

```kdl
kind "Numeric" {
    members {
        "integer"    wasm="i32"
        "integer64"  wasm="i64"
        "float32"    wasm="f32"
        "number"     wasm="f64"
    }
    signature "add" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "sub" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "mul" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "neg" { in { (Self)a }          out { (Self)result } }
}
```

kind は圏論における自然変換を宣言する仕組み。`vectorize` derives が kind の signature から SIMD transition を自動導出する:

```
Numeric kind:
  integer  --- add --> integer       vectorize (関手)
  number   --- add --> number    →   f64x2 --- add --> f64x2
  float32  --- add --> float32       f32x4 --- add --> f32x4
```

`vectorize` は「スカラー圏から SIMD 圏への持ち上げ」を行う関手。instruction solver の BFS がこの持ち上げ先を「より短い経路」として自然に発見する。

### Derives: 既存宣言からの導出

| 戦略 | 説明 | 例 |
|---|---|---|
| `compose` | 既存関数の組み合わせで新関数を定義 | `multiply` + `add` → `sum_of_squares` |
| `binary_accumulate` | 二項演算から反復演算を導出 | `mod_add` → `mod_mul` |
| `conditional` | 条件分岐 | 符号判定で `abs` |
| `host_import` | ホスト環境からの import | WASM ホストから `fp_add` |
| `vectorize` | kind の同型関係から SIMD transition を自動導出 | Numeric kind → `f64x2.add`, `f32x4.mul` |

```kdl
// 合成: 既存関数の組み合わせで新しい関数を定義
derives "math.composed.sum_of_squares" via compose {
    sources "math.arith.multiply" "math.arith.add"
    steps {
        step "math.arith.multiply" input="a,a" output="sq_a"
        step "math.arith.multiply" input="b,b" output="sq_b"
        step "math.arith.add" input="sq_a,sq_b" output="result"
    }
}

// 累積: 二項演算から反復演算を導出 (例: 加算 → 乗算)
derives "math.mod.mod_mul" from "math.mod.mod_add" via binary_accumulate {
    identity 0
    halve "math.bit64.shr1"
    test "math.bit64.test_lsb"
}

// 条件分岐
derives "math.arith.abs" via conditional {
    cond "i32.lt_s(a, 0)"
    then "i32.neg(a)"
    else "a"
}

// ホスト import
derives "math.galois.fp_add" via host_import {
    module "math.galois" name "math.galois.fp_add"
}

// vectorize: kind の同型関係から SIMD transition を自動導出
derives "simd.f64x2" from kind="Numeric" member="number" via vectorize {
    lane-width 2
    target "wasm"
}
derives "simd.f32x4" from kind="Numeric" member="float32" via vectorize {
    lane-width 4
    target "wasm"
}
```

`vectorize` の `target` パラメータは並列化ターゲットを指定する:

| target | 意味 | lane-width の解釈 |
|---|---|---|
| `"wasm"` | WASM SIMD v128 命令 | レーン数 (2 or 4) |
| `"wgpu"` | GPU compute shader dispatch | workgroup_size |
| `"thread"` | CPU thread pool | スレッド数 |

### Import: 外部 bridge の型境界

```kdl
import "neco-vault" {
    procedure "create_sealed_dm" {
        in {
            (str)content
            (bytes)recipient-pubkey
        }
        out {
            (bytes)ciphertext
            (bytes)ephemeral-pubkey
        }
    }
}
```

bridge は Laplan の外にある実装 (Rust crate 等) との接続点。入出力の型だけを宣言し、実装は外部に委譲する。

---

## axiom: 標準ライブラリ

Laplan に組み込みの基本宣言群。全てゼロ外部依存。

> **用語としての axiom:** `axiom` は `.lex` 実装の圏の**外側**にあり、無条件で信頼される宣言を指す。標準ライブラリの基本演算、API の界面外側 (HTTP ヘッダー、ランタイム提供値)、`import` による関手指定 (外部 Rust crate 等) がこれに該当する。axiom の正しさは Laplan の solver が検証する対象ではなく、前提として受け入れられる。数学における公理と同じ位置づけ。

| モジュール | 内容 |
|---|---|
| `math/` | i32 演算, i64 演算, f32 演算, f64 演算, ビット, 論理, 剰余, ガロア体, 行列, 乱数 |
| `kind/` | 同型演算の型束 (Numeric: i32/i64/f32/f64 の算術を統一) |
| `data/` | array, map, bytes, json, cbor, cid, car, encoding (base64/58), kdl, tree |
| `text/` | string, patch, diff, wrap, fuzzy, path |
| `memory/` | linear memory (WASM load/store) |
| `crypto/` | hash (sha256/sha1/hmac), secp256k1, vault |
| `integrity/` | assert |
| `io/` | std (print) |
| `coherence/` | aggregate, batch, dual, inverse, law, compose, accumulate, specialize, conditional, host_import, vectorize, string_derives |

`coherence/` は axiom 間の関係を宣言する。カウント集計の不変条件 (aggregate)、forward/reverse クエリの双対性 (dual)、action/undo の逆元関係 (inverse)、群公理・冪等性・準同型等の代数的法則 (law)、関数合成 (compose)、SIMD vectorize など。

---

## biblio: ドメイン固有ライブラリ

crate のように独立した `.lex` ファイル群。`module.lex` で provides / requires を宣言する。

| biblio | ドメイン |
|---|---|
| `atproto/` | AT Proto (`com.atproto.*`) |
| `bsky/` | Bluesky (`app.bsky.*`) |
| `encrypted-dm/` | 暗号化 DM |
| `bsky-cli/` | CLI コマンド定義 |
| `pds-frontpage/` | PDS フロントページ |

### biblio の構造

例: `biblio/encrypted-dm/`

| パス | 役割 |
|---|---|
| `lex/module.lex` | provides / requires 宣言 |
| `lex/dm.lex` | レコード型・手続き定義 |
| `lex/chain.lex` | handler パイプライン |
| `lex/morphism.lex` | 副作用宣言 |
| `lex/import.lex` | 外部 bridge の型境界 |
| `src/generated/` | codegen 出力 (Rust の場合) |

`module.lex`:
```kdl
module "encrypted-dm" version=1 {
    description "暗号化 DM"
    namespace "ink.illo.dm"

    provides {
        procedure "ink.illo.dm.send"
        procedure "ink.illo.dm.list"
        record "ink.illo.dm.sealed"
    }

    requires {
        capability "access_jwt"
        import "neco-vault" features="nip17"
    }
}
```

---

## 結合系: biblio と libraria

Laplan のプロジェクトは biblio と libraria の二段で構成される。

| Laplan | Cargo | 役割 |
|---|---|---|
| `biblio.lex` | `Cargo.toml` (単一 crate) | 単一パッケージの provides/requires 宣言 |
| `libraria.lex` | `Cargo.toml [workspace]` | 複数パッケージの束ね・面定義 |

各 biblio は自分の provides/requires だけを宣言する。solver が界面を自動接続し、biblio を跨いだ経路を発見する。

libraria は複数の biblio を束ね、面 (face) を定義する。面は「この視点から見たとき、何を生成し、何を前提とするか」を決める。

```kdl
libraria "my-app" {
    members {
        "atproto-client" path="biblio-client"
        "atproto-server" path="biblio-server"
    }
    faces {
        face "client" {
            emit "atproto-client"
            axiom "atproto-server"
        }
        face "server" {
            emit "atproto-server"
            axiom "atproto-client"
        }
    }
}
```

- **emit**: codegen の生成対象。この biblio のコードを出力する
- **axiom**: 信頼する前提。インターフェースのみ参照し、コードは生成しない

client face では server 側が axiom (存在を信頼)、server face では client 側が axiom。同じ libraria でも視点によって solver の探索空間が変わる。

---

## ビルド

### compiler と codegen の組み込みリソース

言語ターゲットの変換規則は、compiler / codegen の**組み込みリソース**として名前空間 `target.*` に配置される。ユーザのプロジェクトには置かない。

| 区分 | 名前空間 | 説明 |
|---|---|---|
| Compiler | `target.binary.wasm` | WASM 命令マッピング |
| Codegen | `target.lang.rust` | Rust の型マッピング・構文パターン・コード生成テンプレート |
| Codegen | `target.lang.python` | Python の同上 |
| Codegen | `target.lang.typescript` | TypeScript の同上 |
| Codegen | ... | 計 21 言語 |

各ターゲットは 3 ファイルで構成される:

| ファイル | 役割 |
|---|---|
| `functor.lex` | 型マッピング |
| `morph.lex` | 構文パターン |
| `type.lex` | 型変換 |

これらは codegen が「この言語をどう書くか」を知るためのルールであり、`rustc` 内部の LLVM バックエンド定義に相当する。

### WASM バイナリビルド

axiom の宣言 + coherence の derives から WASM module を生成する。

> `axiom/*.lex` → **compiler** → `.wasm`

| Compiler ステージ | 入力 | 出力 |
|---|---|---|
| parser | `.lex` | AST |
| ir | AST | 中間表現 |
| compile | IR | WASM (`target.binary.wasm` を使用) |

derives の展開ルール:

- `derives via compose` / `derives via binary_accumulate` → WASM 命令列に展開
- `derives via host_import` → WASM の import section に変換
- `derives via vectorize` → kind の signature × member × lane-width から SIMD transition を自動生成

### コンパイラオプション

| フラグ | 効果 | solver 層 |
|---|---|---|
| `--bake` | module の provides/requires から全到達可能経路を列挙し、frozen chain として WASM 関数を自動生成。ユーザー記述 chain ゼロ、実行時 solve コストゼロ | Layer 1 のみ |
| `--simd` | `--bake` の frozen chain を instruction solver に通し、SIMD 経路がスカラーより短ければ自動置換 | Layer 1 + Layer 2 |
| `--constant-time` | `branching` 属性を持つ transition を solver のフィルタで遮断し、分岐のない経路のみを対象とする | transition フィルタ |
| `--parallel` | 独立 transition を自動検出し、並列実行 DAG を構築。target に応じて WASM threads / wgpu dispatch / thread pool に変換 | Petri net 解析 |

3 つのフラグは直交し、任意に組み合わせ可能:

```bash
# place から SIMD 最適化 + サイドチャネル安全な WASM 関数が自動生成される
laplan compile biblio/encrypted-dm/ --bake --simd --constant-time
```

`--bake` は solver をリンカーにする。シンボル解決を全部コンパイル時に完了する。
`--simd` は BFS の最短経路発見として SIMD を選択する。コスト関数の追加ではない。
`--constant-time` は transition テーブルのフィルタ。solver アルゴリズムは共通。

### 多言語 codegen

axiom + biblio の宣言から、codegen が組み込みターゲット定義を参照して各言語のソースコードを生成する。

> `axiom/*.lex` + `biblio/*.lex` → **codegen** → `src/generated/*`

| ターゲット | 出力 |
|---|---|
| `target.lang.rust` | `src/generated/*.rs` |
| `target.lang.python` | `src/generated/*.py` |
| ... | (全言語同パターン) |

対応言語 (22 ターゲット; 18 active, 3 deferred, 1 removed):

成熟度レベル: L1: 構文 lint 通過, L2: 型コンパイル通過, L3: パッケージビルド通過, L4: ランタイム煙テスト通過。

| 言語 | カテゴリ | 成熟度 |
|---|---|---|
| Rust | システム | L4+ (workspace: 104+ files, WASM 出力, 逆関手完了) |
| Python | スクリプト | L4 |
| Go | システム | L4 |
| Ruby | スクリプト | L4 |
| Kotlin | JVM / モバイル | L4 |
| Java | JVM | L4 |
| Lua | スクリプト | L4 |
| TypeScript | Web | L3 |
| Swift | モバイル | L3 |
| C# | .NET | L3 |
| Dart | モバイル | L3 |
| Zig | システム | L3 |
| Clojure | JVM | L3 |
| Haskell | 関数型 | L2 |
| C++ | システム | L2 |
| D | システム | L2 |
| JavaScript | Web | L1 |
| PHP | スクリプト | L1 |
| OCaml | 関数型 | deferred |
| Elixir | 関数型 | deferred |
| Gleam | 関数型 | deferred |

### biblio の出力先

codegen のターゲットによって出力先が異なる。Rust ターゲットの例 (`biblio/encrypted-dm/`):

| パス | 役割 | ソース |
|---|---|---|
| `lex/*.lex` | 宣言 | 入力 (ソース) |
| `src/generated/mod.rs` | モジュールルート | codegen 出力 |
| `src/generated/handler.rs` | handler 実装 | codegen 出力 |
| `src/bridge.rs` | bridge 実装 | ソース |
| `src/lib.rs` | `generated/` を bind → crate として機能 | ソース |
| `Cargo.toml` | crate マニフェスト | ソース |

Rust の場合、`src/generated/` + `src/lib.rs` で crate として機能する。WASM バイナリの場合は `target/`（Cargo と同様）。

---

## 推奨ワークフロー

1. **Place を書く** ... 型の定義 (Lexicon JSON からの変換も可)
2. **Solver に渡す** ... 「何を持っていて、何が欲しいか」の指定
3. **収束を見る**
   - 1 経路 → 完成
   - 複数経路 → 制約不足
   - 到達不能 → missing facts を確認
4. **足りなければ足す** ... morphism, capability, 数行の axiom の追加
5. **1 に戻る**

AT Protocol の Lexicon (約 100 NSID) をそのまま読み込み、リクエストヘッダーの capability を数行足しただけで、solver が全経路を自動解決した実績がある。Lexicon の設計者は Petri net のことを考えていなかったが、API 定義として型と入出力を誠実に書いていれば、それだけで経路は収束する。

**place を誠実に書けば、経路は勝手に生まれる。**

---

## 設計原則

1. **型が根源。** 型があれば Petri net が生まれ、経路の問いが成立する。関数は経路の制約であって、起点ではない。
2. **宣言が正本。** コードは宣言から生成される。ユーザー記述コードの存在は宣言の不足を意味する。
3. **個は有限、全体はチューリング完全。** solver の探索空間は有限で必ず停止し、全経路の列挙と診断が可能。各 bridge (Rust crate, WASM, HTTP API) も有限のインターフェースを持つ。しかしそれらの積み上げの可能性空間はチューリング完全に到達する。個々が探索可能だからこそ、全体が健全にチューリング完全になる。
4. **ゼロ外部依存。** axiom の bridge backing (neco-crates) は全て pure Rust。環境を選ばない。
5. **相互変換。** `.json ↔ .kdl ↔ .lex` は `xrpc=` 付きなら完全可逆。型名も双方向変換 (`str` ↔ `string`)。
6. **Petri net 経路解決。** 「持っているもの」と「必要なもの」を宣言すれば、solver が合成経路を発見する。
7. **最適化は経路発見の帰結。** SIMD はコスト関数の追加ではなく、instruction solver の BFS 最短経路発見の結果。定数時間実行は transition フィルタの結果。並列化は独立 transition の自動検出の結果。最適化パスを追加するのではなく、Petri net の構造的性質を利用する。
