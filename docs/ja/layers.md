# .lex トップノードと層分類

## 序列

```
type    型の存在
lex     Lex₀  型の構造 (入出力の定義)
rule    Lex₁  型間の規則 (solver 探索対象)
─── solver boundary ───
func    Lex₂  構造制約 (solver の判断材料)
pkg     Lex₃  パッケージ管理
```

各層は下の層を信頼する公理として扱う。prefix は省略可能 (既存互換)。

Lex₁ と Lex₂ の間の **solver boundary** が収束性の鍵: solver は Lex₁ だけを探索し、Lex₂ は枝刈り・同一視・導出の根拠として参照する。この境界がなければ探索空間が爆発する。

## type: 型

型そのものの定義。全ての層の基盤。

| 記法 | 例 |
|---|---|
| `type` | `type i32` |

## lex (Lex₀): 型の構造

Petri net の place。型と入出力の構造を定義する。

| 記法 | 例 | 説明 |
|---|---|---|
| `lex` | `lex i32.add version=1 { ... }` | 型の入出力定義 |

## rule (Lex₁): 規則

Petri net の transition。**solver が探索する。**

| 記法 | 省略形 | 説明 |
|---|---|---|
| `rule` | | 要件と成果物 |
| `rule.const` | `const` | 定数束縛。初期マーキング (`() → T`)。再代入不可 |
| `rule.assign` | `assign` | 変数束縛。token flow (`() → T`)。再代入可 |
| `rule.inverse` | `inverse` | 逆操作 (枝刈り) |
| `rule.derives` | `derives` | 既存規則からの導出 |
| `rule.chain` | `handler`/`chain` | 手動経路指定 |
| `rule.refinement` | `refinement` | 既存 lex への制約追加 |

## func (Lex₂): 構造制約

Lex₁ の規則を前提として構造的制約を置く。**solver は探索しない。** 枝刈り・同一視・導出規則として利用される。

| 記法 | 省略形 | 説明 |
|---|---|---|
| `func` | | 言語変換規則 |
| `func.family` | `family` | 型族 (同型演算を共有する閉じた型の集合。vectorize/product の導出元) |
| `func.law` | `law` | 代数的法則 |
| `func.dual` | `dual` | 双方向クエリの対称性 |
| `func.invariant` | `invariant` | 不変量 (count 整合性等) |

## pkg (Lex₃): パッケージ管理

Lex₂ 以下を束ねてパッケージ化する。外部との接続を管理。

| 記法 | 省略形 | 説明 |
|---|---|---|
| `pkg.import` | `import` | 外部実装の型境界 |
| `pkg.cratis` | `cratis` | パッケージ / ワークスペース宣言 |

`cratis` は単体パッケージとワークスペースを兼ねる。`members` の有無で判定:

```kdl
// 単体パッケージ
cratis "encrypted-dm" version=1 {
    provides { ... }
    requires { ... }
}

// ワークスペース (members がある)
cratis "my-app" {
    members {
        "atproto-client" path="client/cratis.lex"
        "atproto-server" path="server/cratis.lex"
    }
    faces {
        face "client" { emit "atproto-client"; axiom "atproto-server" }
        face "server" { emit "atproto-server"; axiom "atproto-client" }
    }
}
```

## 判定基準

| 質問 | 層 |
|---|---|
| 型の存在宣言か? | type |
| 入出力構造の定義か? | Lex₀ (lex) |
| solver が遷移として探索するか? | Lex₁ (rule) |
| solver は探索せず、判断材料として使うか? | Lex₂ (func) |
| パッケージ/外部接続の管理か? | Lex₃ (pkg) |

## axiom/ ディレクトリとの対応

```
axiom/
  i32/            Lex₀ + Lex₁ (型定義 + 演算)
  str/            Lex₀ + Lex₁
  crypto/         Lex₀ + Lex₁
  category/       Lex₂ (compose, dual, lift)
  algebra/        Lex₀ + Lex₁ + Lex₂ (演算 + family)
  target/         Lex₂ (言語/バイナリ/バインディング変換規則)
    lang/           mapping.lex (型対応表) + rule.lex + type.lex
    binary/
    bind/
```

ひとつのディレクトリが複数の層を含むことがある。層の分離はディレクトリではなくノード prefix で判定する。
