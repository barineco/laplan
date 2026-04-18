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
        boundary { from "cratis-a" to "cratis-c" via "http" }
    }
}
```

| フィールド | 意味 |
|---|---|
| `emit` | この face から生成するターゲット cratis |
| `axiom` | この face で axiom (公理) として扱う cratis |
| `capability` | この face の初期 capability。workspace モードの solver が初期 marking に展開する |
| `bind` | この face のバインディング言語 (optional) |
| `boundary` | cratis 間の境界通信仕様 |

`client { ... }` / `server { ... }` は `face "client" { ... }` / `face "server" { ... }` の省略形です。

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
