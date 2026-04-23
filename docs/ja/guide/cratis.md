# cratis の書き方

cratis は laplan のパッケージ / ワークスペース宣言です。`.lex` 群を束ね、外部実装との境界 (provides / requires / faces) を管理します。

## 単体 cratis

```kdl
cratis "encrypted-dm" version=1 {
  provides {
    procedure "com.example.dm.send"
    procedure "com.example.dm.receive"
  }
  requires {
    axiom "str.concat"
    axiom "crypto.encrypt"
  }
}
```

| ブロック | 意味 |
|---|---|
| `provides { procedure/query/record/... }` | この cratis が外部に提供する宣言 |
| `provides { axiom ... }` | 提供する axiom |
| `requires { axiom ... }` | 依存する axiom |

## workspace cratis

`members` を持つ cratis はワークスペースです。複数の子 cratis を束ね、client / server など face で分離します。

```kdl
cratis "my-app" {
  members {
    "atproto-client" path="client/cratis.lex"
    "atproto-server" path="server/cratis.lex"
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

## CratisSource の 3 形式

`members` の各エントリは以下のいずれかで指定します。

| 形式 | 記法 | 用途 |
|---|---|---|
| Path | `path="client/cratis.lex"` | ローカルの相対パス |
| Builtin | `from="axiom/crypto"` | laplan 同梱の組込 cratis |
| GitHub | `from="github:owner/repo/path" hash="..."` | GitHub 取得 (filesystem feature 必須) |

これら 3 種類のソース指定をサポートしています。

## face

```kdl
faces {
  face "client" {
    emit "cratis-a"
    emit "cratis-b"
    axiom "cratis-c"
    bind "typescript"
    boundary {
      "com.example.account.create" emit="server" message="CreateAccountRequest"
    }
  }
}
```

| フィールド | 意味 |
|---|---|
| `emit` | この face から生成するターゲット cratis |
| `axiom` | この face で axiom (公理) として扱う cratis |
| `capability` | この face の初期 capability。workspace モードの solver が初期 marking に展開する |
| `trust` | 相手 face の信頼水準。省略時は `full` |
| `bind` | この face のバインディング言語 (optional) |
| `boundary` | cratis 間の境界通信仕様 |

face 名は新形式 `face "name" { ... }` で任意に付けられます。現在の制約は次のとおりです。

| ルール | 内容 |
|---|---|
| 空文字列不可 | `face "" { ... }` は不可 |
| 使用可能文字 | 英数字、`-`、`_` のみ |
| 重複不可 | 同一 cratis 内で同じ face 名は 1 回だけ |

予約名として `client`、`server`、`pds`、`appview`、`bgs` を使えます。これらは将来や runtime で特別な意味を持ち得る名前ですが、parse 層では他の任意名と同じく受理されます。

`client { ... }` / `server { ... }` は `face "client" { ... }` / `face "server" { ... }` の省略形です。この legacy 省略形は client / server にだけ使えます。任意名 face は必ず `faces { face "browser" { ... } }` の新形式で書いてください。

### trust

`trust="lexicon-only"` を付けた face は stub face として扱われます。

```kdl
face "bsky-official-pds" trust="lexicon-only" {
  axiom "com.atproto"
  capability "pds-endpoint"
}
```

- `full` (default): 通常の laplan-aware face。rule と内部 path まで solver が追う
- `lexicon-only`: 内部 path を追わず、Lexicon で宣言された endpoint output を無条件に `produces` に持つとみなす
- stub face との `message` mismatch は warning に降格される
- synthesis は stub face 向け endpoint に response validator API を生成する。runtime で受信 body が Lexicon output と合わなければ `MismatchError` を返す

## boundary

`boundary` ブロックは、face を跨いで公開する endpoint と、その遷移先 face を宣言します。

```kdl
boundary {
  "com.example.account.create" emit="server" message="CreateAccountRequest"
  "com.example.account.create" emit="client" message="CreateAccountResponse"
}
```

| 属性 | 意味 |
|---|---|
| `emit` | 遷移先の face 名。`wasm` のような binary target も指定できます |
| `message` | その境界で受け渡す型名 (optional) |

`message` を省略した rule は従来どおり emit だけを使います。`message` を指定した rule は workspace solve の前に静的に検証され、未知の型名、同一 pattern の競合があると solve は中断します。request / response の不一致は通常 face 同士では error ですが、stub face を相手にすると warning へ降格します。

## 判定基準

| 質問 | 所属 |
|---|---|
| 型の存在宣言か? | type |
| 入出力構造の定義か? | Lex₀ lex (lexicon) |
| solver が遷移として探索するか? | Lex₁ solve (rule) |
| solver は探索せず、判断材料として使うか? | Lex₂ constrain |
| パッケージ / 外部接続の管理か? | Lex₃ package (cratis / import) |

詳細は [reference/layers.md](../reference/layers.md) 。

## cratis ファイルの配置

```
project/
├── cratis.lex               # ワークスペース
├── client/
│   ├── cratis.lex           # face "client" 用の単体 cratis
│   ├── rule.lex
│   └── ...
└── server/
    ├── cratis.lex
    ├── rule.lex
    └── ...
```

パーサは `cratis` / `members` / `faces` / `client` / `server` ブロックを解釈します。

## cratis.lex の配置状況

`axiom/` 直下の各カテゴリ (`i32`, `crypto`, `algebra` など) には単体 `cratis.lex` が配置され、`provides { axiom ... }` / `requires { axiom ... }` で package 境界を明示します。`axiom/target/` は親 workspace として `lang` / `binary` / `bind` の group cratis を束ねます。

`provides { axiom ... }` は package metadata です。endpoint の読み込みとは独立しています。cratis.lex の member 解決と package ディレクトリ内の peer `.lex` 読み出しが行われますが、`derives` / `const` / `rule` の自動展開は行われません。

axiom の全体像は [reference/layers.md](../reference/layers.md) の axiom テーブル、および [reference/axiom-*](../INDEX.md) 群を参照してください。
