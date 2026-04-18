# 言語の追加

新しい言語への SDK 生成を追加する手順です。`axiom/target/lang/{lang}/` に宣言的なテンプレートを置けば、`GenericBackend` が自動で取り込みます。Rust 本体に言語固有コードを書く必要はありません (特殊ケースを除く)。

## ディレクトリ構造

```
axiom/target/lang/{lang}/
├── mapping.lex    # 型・構文・制御・handler テンプレート (必須)
├── morph.lex      # 射 (axiom) の言語別実装 (必須)
├── type.lex       # 追加の型宣言 (任意)
└── `mapping.lex` の `cli { ... }`  # CLI synthesis 用 (任意)
```

## mapping.lex の書き方

Rust の例:

```kdl
extension "rs"

keywords strict "type" "self" "fn" "struct" "enum" "trait" "impl" "pub" "use" ...
keywords builtins "String" "Result" "Type" "Object"
file-collisions "import" "package" "module" "type" "mod" "nul" "con" "prn" "aux"
allow-keyword-fields #false
identifier-escape prefix="r#"

syntax {
    product {
        visibility "pub"
        attribute "#[derive(Debug, Clone, PartialEq)]"
        keyword "struct"
        open "{"
        close "}"
        field-format "    pub {name}: {type},"
    }
    sum { /* ... */ }
    alias { format "pub type {Name} = {type};" }
    import-format "use crate::{gen}::{path::}::{Name};"
}

control {
    if-open "if {cond} {"
    else-open "} else {"
    if-close "}"
    for-open "for {var} in &{collection} {"
    for-close "}"
    fn-open "pub fn {name}({params}) -> {return_type} {"
    fn-close "}"
    module-open ""
    module-close ""
}

variable {
    binding "let {name} = {value};"
    mutable-binding "let mut {name} = {value}.to_owned();"
    assign "{target} = {value};"
    return "return {value};"
}

handler {
    handler-open "pub trait {HandlerName}: Send + Sync {"
    handler-close "}"
    method "    fn handle({MethodParams}) -> {ReturnType};"
    // ...
}
```

### 必須セクション

| セクション | 目的 |
|---|---|
| `extension` | 出力拡張子 |
| `keywords strict` | 予約語 (識別子衝突検査) |
| `keywords builtins` | 組込型 |
| `syntax { product, sum, alias }` | 型宣言テンプレート |
| `control` | 制御構文 |
| `variable` | 束縛・代入・return |
| `handler` | endpoint ハンドラ trait |

### 任意セクション

| セクション | 目的 |
|---|---|
| `functional` | Lex₁ パス (関数型言語のみ)。let-in / match / lambda / fold 等 |
| `lowering` | `fst` / `snd` / `from-maybe` 等の lowering テンプレート |
| `bindings` | 外部ライブラリマッピング |
| `stub-template` | スタブコード |
| `effect-*` | effect 型と値の表現 |

### template 変数

| 変数 | 意味 |
|---|---|
| `{name}`, `{Name}`, `{NAME}` | 小文字 / PascalCase / 大文字 |
| `{type}`, `{return_type}`, `{params}` | 型とシグネチャ |
| `{cond}`, `{var}`, `{collection}`, `{value}`, `{target}` | 制御構文の埋め込み |
| `{gen}`, `{Gen}` | `generated-subdir` の basename (小文字 / PascalCase) |
| `{path::}`, `{path.}`, `{path/}` | モジュールパスの区切り文字 (`::`, `.`, `/`) |

## morph.lex

axiom の射 (例: `i32.add`) を言語で実装する形を宣言します。

```kdl
morph "i32.add" {
    inline "({a} + {b})"
}

morph "str.concat" {
    inline "format!(\"{}{}\", {a}, {b})"
}
```

テンプレート変数には引数名 (`a`, `b`, ...) と返値変数が埋め込まれます。

## type.lex

言語固有の追加型宣言。空でも可。

## 関数型言語の場合

Haskell / OCaml / Gleam / Elixir のように Lex₁ パスを使う言語では、mapping.lex に `functional {}` セクションを追加します。

```kdl
functional {
    let-in "let {bindings} in {body}"
    match "case {target} of { {branches} }"
    lambda "\\{params} -> {body}"
    // ...
}
```

`has_functional_templates()` が `functional {}` の有無で Lex₁ / Lex₂ パスを自動分岐します。

## テスト追加

### runtime_emit 回帰テスト

```bash
cargo test -p laplan-synthesis --lib runtime_emit
```

新言語のテンプレートで既存 axiom をレンダリングし、期待出力と一致するか検証します。

### Lex₁ パスの場合

```bash
cargo test -p laplan-synthesis --lib template_engine_fn
cargo test -p laplan-synthesis --lib runtime_program_fn
cargo test -p laplan-synthesis --lib write_runtime_resolve_fn
```

### ビルド検証

```bash
cargo check -p laplan-synthesis
cargo check -p laplan-synthesis --no-default-features
```

## 確認リスト

1. `axiom/target/lang/{lang}/mapping.lex` を配置
2. `axiom/target/lang/{lang}/morph.lex` で axiom 実装を宣言
3. `all_mapping_names()` が新言語を返すか確認
4. `generic_backend_for_target("{lang}")` が `Some` を返すか確認
5. 回帰テスト (`runtime_emit`) の期待出力を追加
6. docs を更新:
   - [reference/target-languages.md](../reference/target-languages.md) に Capability level を追記
   - 必要に応じて下流プロジェクト (neco-atproto 等) の `guide/{lang}.md` を更新

## 特殊ケース

以下の対象は `GenericBackend` 外で追加コードが必要です。

| 対象 | 追加先 |
|---|---|
| WASM binary | `compiler/synthesis/src/wasm_emit.rs`, `wasm_lower.rs` |
| WGSL shader | `compiler/synthesis/src/wgsl_emit.rs` |
| Python binding | `compiler/synthesis/src/bind_python.rs` |
| TypeScript binding | `compiler/synthesis/src/bind_typescript.rs` |
| サーバ実装 (atproto-server feature) | `compiler/synthesis/src/bind_server.rs`, `server_output.rs` |

これらは言語テンプレートではなく emit 層の対応になります。詳細は [architecture/synthesis.md](../architecture/synthesis.md) 。
