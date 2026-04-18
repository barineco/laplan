# .lex トップノードと層分類

## 序列

```
type      型の存在
lexicon   Lex₀ (lex)        型の構造 (入出力の定義)
morph     Lex₁ (solve)      型間の射 (solver 探索対象)
─── solver boundary ───
func      Lex₂ (constrain)  構造制約 (solver の判断材料)
pkg       Lex₃ (package)    パッケージ管理
```

各層は下の層を信頼する公理として扱います。prefix は省略可能です (既存互換)。

Lex₁ と Lex₂ の間の **solver boundary** が収束性の鍵です。solver は Lex₁ だけを探索し、Lex₂ は枝刈り・同一視・導出の根拠として参照します。この境界がなければ探索空間が爆発します。

## type: 型

型そのものの定義。全ての層の基盤。

| 記法 | 例 |
|---|---|
| `type` | `type i32` |

## lexicon (Lex₀ lex): 型の構造

Petri net の place。型と入出力の構造を定義する。

| 記法 | 例 | 説明 |
|---|---|---|
| `lexicon` | `lexicon i32.add version=1 { ... }` | 型の入出力定義 |

## rule (Lex₁ solve): 射

Petri net の transition。**solver が探索する。**

| 記法 | 省略形 | 説明 |
|---|---|---|
| `rule` | `rule` | 要件と成果物 |
| `morph.const` | `const` | 定数束縛。初期マーキング (`() → T`)。再代入不可 |
| `morph.assign` | `assign` | 変数束縛。token flow (`() → T`)。再代入可 |
| `rule.inverse` | `inverse` | 逆射 (枝刈り) |
| `rule.derives` | `derives` | 既存射からの導出 |
| `morph.chain` | `handler`/`chain` | 手動経路指定 |
| `morph.refinement` | `refinement` | 既存 lexicon への制約追加 |

## constrain (Lex₂): 構造制約

Lex₁ の射を前提として構造的制約を置く。**solver は探索しない。** 枝刈り・同一視・導出規則として利用される。

| 記法 | 省略形 | 説明 |
|---|---|---|
| `func` | `mapping` | 言語変換規則 |
| `func.family` | `family` | 型族 (同型演算を共有する閉じた型の集合。vectorize/product の導出元) |
| `func.law` | `law` | 代数的法則 |
| `func.dual` | `dual` | 双方向クエリの対称性 |
| `func.invariant` | `invariant` | 不変量 (count 整合性等) |

## package (Lex₃): パッケージ管理

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
| 入出力構造の定義か? | Lex₀ lex (lexicon) |
| solver が遷移として探索するか? | Lex₁ solve (rule) |
| solver は探索せず、判断材料として使うか? | Lex₂ constrain |
| パッケージ/外部接続の管理か? | Lex₃ package |

## axiom/ ディレクトリとの対応

```
axiom/
  i32/            Lex₀ + Lex₁ (型定義 + 演算)
  str/            Lex₀ + Lex₁
  crypto/         Lex₀ + Lex₁
  category/       Lex₂ + Lex₃ (`cratis.lex` が package root、compose/dual/lift は Lex₂)
  algebra/        Lex₀ + Lex₁ + Lex₂ (演算 + family)
  resolver.lex    Lex₁ (FnExpr KDL。runtime resolver 7 関数)
  target/         Lex₂ + Lex₃ (`cratis.lex` が workspace root、配下の `lang/` `binary/` `bind/` も各 `cratis.lex` を持つ)
    lang/           mapping.lex (型対応表) + morph.lex + type.lex
    binary/
    bind/
```

`resolver.lex` は FnExpr を KDL で直接記述する .lex バリアントです。通常の .lex が lexicon / rule / mapping 等の宣言を扱うのに対し、resolver.lex は `fn` ノードで Lex₁ の関数定義を記述します。`parse_resolver_lex()` が KDL → `Vec<FnDef>` に変換し、lowering 経由で全言語に resolver コードを生成します。詳細は [architecture/ir.md](../architecture/ir.md) の「resolver.lex」節を参照。

各 `axiom/*/cratis.lex` は Lex₃ の package root で、`axiom/target/cratis.lex` は `lang` `binary` `bind` を束ねる workspace root になります。
ここでの `provides` は package metadata です。endpoint の読み込みとは独立しています。
ひとつのディレクトリが複数の層を含むことがあります。層の分離はディレクトリではなくノード prefix で判定します。
