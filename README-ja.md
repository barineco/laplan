# Laplan: Lexicon as Programming Language

[English](README.md) | [docs](docs/ja/INDEX.md)

KDL 形式で型と関係を宣言し、Petri net 経路解決で合成する型型プログラミング言語です。

## 何を持っていて、何が欲しいのか

API は入力を受け取り、出力を返します。この「入力 → 出力」の関係は型の写像です。getProfile は actor を受け取り profile を返す。createSession は credential を受け取り access_jwt を返す。

これらの写像を繋げれば、「今持っているもの」から「欲しいもの」への経路が生まれます。Laplan はこの経路探索を自動化します。

```rust
// 従来: 認証→リクエスト→エラー処理→リトライを手で書く
let session = client.create_session(&credentials).await?;
let did = client.resolve_handle(&session.access_jwt, &handle).await?;
let profile = client.get_profile(&session.access_jwt, &did).await?;
// access_jwt 期限切れ? → 手動で refresh して retry...
```

```rust
// Laplan: 欲しいものを言う
client.fetch_goal("datum:profileViewDetailed", &params)
```

1 行で書けます。認証、トークン更新、ハンドル解決、リクエスト送信の全ステップを solver が自動的に発見・実行します。

必要なのは、各 API が何を必要とし (`requires`)、何を生み出すか (`produces`) を宣言することだけです:

```kdl
rule "com.atproto.server.createSession" {
    produces capability="access_jwt"
    produces capability="refresh_jwt" ttl=86400
}

rule "app.bsky.actor.getProfile" {
    requires capability="access_jwt"
    requires input="actor"
    produces output="profile"
}
```

solver はこの宣言を [Petri net](https://ja.wikipedia.org/wiki/%E3%83%9A%E3%83%88%E3%83%AA%E3%83%8D%E3%83%83%E3%83%88) として読み取り、BFS (幅優先探索) で到達可能な経路を発見します。手続きを書く必要はありません。明示したい場合は chain で順序を固定できます。

---

## 何が嬉しいのか

### 実行の暗黙順序を無くす

手続きの暗黙順序がありません。「認証の前にリクエストを送る」「トークン更新を忘れる」「API 呼び出しの順序を間違える」といったバグは、solver が経路を自動決定するため構造的に発生しません。手動で書いた呼び出し順序のバグは、手動で書かなければ生まれません。

### 到達不能なリクエストを送らない

solver は実行前に到達可能性を計算します。「このリクエストは認証トークンがないから到達不能」と判定されたら、HTTP リクエストを送信しません。client と server を接続した系で solver を動かせば、そもそも到達不可能なリクエストを試みないため、ネットワーク負荷が軽く、分散システムに向いています。

到達不能の場合、solver は何が足りないかを [5 段階で分類](docs/ja/architecture/solver.md)して報告します:

| 診断 | 意味 |
|---|---|
| **Matched** | 到達可能、経路あり |
| **MultiPath** | 複数経路あり (制約不足の合図) |
| **PrunedByBoundary** | 認証・権限・データ不足で遮断 |
| **PrunedByRefinement** | refinement で追加した制約 (認証トークン等) により遮断 |
| **MissingFact** | 必要な fact が存在しない |

### コードが劇的に短くなる

宣言だけ書けば、手続き・エラーハンドリング・リトライロジック・認証フローの全てが solver の経路発見から導出されます。chain を書いてもよいですが、書かなくても構いません。

### 21 言語間で意味を保存する変換

Laplan は 21 言語に対して双方向の変換を持ちます:

- **[synthesis](docs/ja/architecture/synthesis.md)**: Lex IR → 21 言語のソースコード生成
- **[inverse](docs/ja/architecture/compiler.md)**: 21 言語のソースコード → Lex IR への逆変換

この 2 つを組み合わせると、既存コードを入力として別の言語に変換できます:

```
Source(Rust) ──inverse──→ Lex IR ──synthesis──→ Source(TypeScript)
```

従来の transpiler (C2Rust, j2objc 等) は言語ペアごとに専用の変換器を書く N^2 構造ですが、Laplan は Lex IR をハブとした N×2 構造をとり、mapping.lex 1 枚で 1 言語を追加できます。

変換の正しさの基準は構文の一致ではなく **alpha 等価** (束縛変数のリネームを除いて論理的に同一) です。21 言語 × 9 関数の roundtrip テストで検証しています。

| 言語 | カテゴリ | | 言語 | カテゴリ |
|---|---|---|---|---|
| Rust | システム | | Swift | モバイル |
| C++ | システム | | Dart | モバイル |
| Zig | システム | | Kotlin | JVM / モバイル |
| Go | システム | | Java | JVM |
| D | システム | | C# | .NET |
| TypeScript | Web | | Clojure | JVM |
| JavaScript | Web | | Haskell | 関数型 |
| Python | スクリプト | | OCaml | 関数型 |
| Ruby | スクリプト | | Elixir | 関数型 |
| PHP | スクリプト | | Gleam | 関数型 |
| Lua | スクリプト | | | |

### 依存の自動解決

同じ capability に対して複数言語の実装を接続できます:

```kdl
import "neco-vault" {
    procedure "verify_signature" {
        in { (bytes)message; (bytes)signature; (bytes)pubkey }
        out { (bool)valid }
    }
}
```

TypeScript 向けに生成すれば npm パッケージ側の依存だけが残り、Rust 向けに生成すれば neco-crates 側の依存だけが残ります。到達不能な依存は solver が自動的に除去するため、言語ペアごとの import テーブルを手で書く必要はありません。

### スキーマ進化時の影響診断

API が変わったとき、solver が「どの経路が壊れたか」を即座に報告します。到達不能になった transition や新たに必要になった capability が自動検出されるため、呼び出し元を検索して回る必要はありません。

### 自動リカバリ経路

`access_jwt` が期限切れになった場合、solver は「何が足りないか」から逆算し、`refreshSession` 経路を自動挿入します。リトライロジックを手で書く必要はありません。

### SIMD / GPU / 並列化の自動最適化

solver は SIMD 経路とスカラー経路を同一テーブルで探索し、最短経路として SIMD を自動選択します。最適化パスを別途追加するのではなく、Petri net 上の最短経路発見の帰結です。詳細は [二層 solver](docs/ja/architecture/solver.md) を参照してください。

```bash
laplan compile cratis/encrypted-dm/ --bake --simd --constant-time
```

`--bake` (全経路の事前列挙)、`--simd` (SIMD 自動選択)、`--constant-time` (分岐なし経路のみ) は直交フラグで、任意に組み合わせられます。

### 形式検証された健全性

solver の到達可能性、制約による到達性の変化、注釈が型から復元できないことは [lean-lexicon](https://github.com/barineco/lean-lexicon) で Lean 4 / Mathlib により形式検証されています (zero sorry)。

---

## 宣言の構造

[4 層の制約](docs/ja/reference/layers.md)を積み重ねます。上ほど solver に委ね、下ほどプログラマが制御します。**上だけで十分なら下は省略できます。**

| 層 | 宣言する内容 | スタイル |
|---|---|---|
| **型** (place) | 何があるか | 最も宣言的 |
| **rule** (transition) | 何から何へ行けるか | |
| **関数** | 何が入り何が出るか | |
| **chain** | どの順で通るか | 最も命令的 (省略可能) |

型を書けば solver が経路を見つけます。足りなければ rule で方向を与え、関数で入出力を絞り、chain で順序を固定します。高レイヤー (API) では rule だけで済むことが多いです。

詳細は [docs](docs/ja/INDEX.md) を参照してください。

---

## 3 つの形式

同じ API 定義 (`app.bsky.actor.getProfile`) を 3 つの形式で書けます:

| 形式 | 用途 | 相互変換 |
|---|---|---|
| `.json` (Lexicon JSON) | AT Protocol 公式原本 | `.json ↔ .kdl ↔ .lex` 完全可逆 |
| `.kdl` (Lexicon KDL) | JSON の人間可読写像 | |
| `.lex` (Laplan KDL) | 計算言語。rule / chain / law を記述可能 | |

AT Protocol の Lexicon (約 100 NSID) をそのまま読み込み、リクエストヘッダーの capability を数行足しただけで solver が全経路を自動解決した実績があります。

---

## ツール

| ツール | 状態 | 説明 |
|---|---|---|
| `laplan` CLI | 利用可能 | compile, synthesis, inverse, solve, diagnose |
| VSCode 拡張 (linter) | 利用可能 | `.lex` ファイルのリアルタイム診断 |
| グラフ可視化 | 開発中 | Petri net / 依存グラフのノードエディタ |
| WASM パッケージ | 利用可能 | ブラウザ / Node.js での solver 実行 |

---

## Solver の設計判断

### なぜ最短経路か

solver は BFS で最短経路を返します。これは実装の都合ではなく、意味論的な選択です。

短い経路は少ない requires で到達できます。つまり最短経路は「最も仮定が少ない解」です。逆に、長い経路は多くの中間状態を通過するため、より specific な制約を必要とします。経路長が情報量の指標になっています。

制約が不足していると solver は同距離の複数経路を返します ([MultiPath 診断](docs/ja/architecture/solver.md))。これは「最短経路で区別がつかない」という設計フィードバックです。経路を一意にするには、より具体的で意味的な制約を追加します。型の精密化による解消の具体例は [solver と型規律](docs/ja/case/solver-type-discipline.md) を参照してください。

- **中間の token を追加**: requires を足せば、正しい最短経路が自然と一意に定まります
- **chain で固定**: solver の枠外で順序を直接指定します

どちらの場合も曖昧な状態は残りません。前者は制約を足して最短経路を変え、後者は solver に任せず手動で確定します。

### 停止性と計算の境界

solver は必ず停止します。各 rule は単一の遷移であり、反復は fold や [safe recursive](docs/ja/reference/axiom/algebra.md) (bounded / decreasing) で表現されます。実用上のほとんどの場面ではこれで十分です。

収束が不明な計算 (数値解析の反復収束、未解決問題等) に対しては 2 つの手段があります:

| 手段 | solver の扱い | 用途 |
|---|---|---|
| unsafe recursive | ラベルとして認識するが経路探索の対象にしない (Lex₂) | 収束条件が自明でない反復 |
| 外部 bridge (import) | 型境界だけを宣言し、内部実装は信頼する | 外部ライブラリへの委譲 |

系を大きくすればチューリング完全に漸近しますが、solver が探索する範囲は常に有限で停止します。詳細は [solver](docs/ja/architecture/solver.md) を参照してください。

---

## 設計原則

1. **型が根源。** 型があれば Petri net が生まれ、経路の問いが成立します
2. **宣言が正本。** コードは宣言から生成されます。手書きコードの存在は宣言の不足を意味します
3. **最適化は経路発見の帰結。** SIMD、定数時間実行、並列化は Petri net の構造的性質から導出されます

---

宣言の書き方、アーキテクチャ、axiom リファレンス、言語追加ガイドは [docs](docs/ja/INDEX.md) にまとめています。
