# 置換記法: mapping.lex の template 仕様

`axiom/target/lang/<lang>/mapping.lex` は、言語別の emit と invert の双方を同じ template で定義します。forward (IR からソースコードを生成) では template 内の placeholder を値で展開し、inverse (ソースコードから IR を復元) では同じ template を照合パターンとして用いて source から値を取り出します。

この双方向性は、本書で扱う 2 系統の記法で実現しています。

| 記法 | 役割 | 所在 |
|---|---|---|
| `{name}` placeholder | 値の展開と捕捉。forward で変数を埋め、inverse で source の断片を capture する | 全言語の mapping.lex で汎用 |
| `«directive»` directive | block 境界を構文的に示す semantic marker。forward 側の出力からは除去され、inverse 側のみが解釈する | 現状は python の `«dedent»` のみ |

以降、各記法の詳細と、forward / inverse の対応関係を順に扱います。

## 1. `{name}` placeholder

template 中の `{name}` は、forward と inverse の両方で使う変数参照です。

### 基本形

`{name}` は識別子 1 つに対応します。`name` の部分は template の定義位置に応じて、IR のフィールド名 (`name`, `type`, `value`) または構造名 (`Name` で表示名相当) を指します。

forward では IR の値が展開されます。

```kdl
binding { emit "let {name} = {value};" }
```

この `binding` template は、rust の mapping.lex で使われている `let` 束縛の emit 定義です (`axiom/target/lang/rust/mapping.lex:64`)。`{name}` と `{value}` には IR の該当フィールドが展開され、`let x = 1;` のような出力になります。

inverse では、同じ template が source の照合パターンとして機能し、`{name}` / `{value}` 位置の source 断片を capture します。照合した識別子と式が IR に戻されます。

### 大文字小文字の区別

`{name}` と `{Name}` は別物です。mapping.lex 側で、表示名と型名の 2 種類を分離するための慣習があります。

| 記法 | 典型的な意味 |
|---|---|
| `{name}` | フィールド名や変数名 (小文字始まり) |
| `{Name}` | 型名や構造名 (大文字始まり) |

言語の命名規則 (PascalCase / snake_case) に応じて、どちらを使うかが template ごとに決まります。

### list / block の capture

単一値ではなくリスト全体を受け取る placeholder もあります。以下は template ごとに役割が定まっており、template が `{args}` / `{params}` / `{body}` のような名前を書けば、対応する構造が正しく展開・capture されます。

| 記法 | 対応する構造 |
|---|---|
| `{args}`, `{params}` | 引数列やパラメータ列 (区切り文字は言語依存) |
| `{body}`, `{then_body}`, `{else_body}` | 文列の block |
| `{arms}` | match の arm 列 |
| `{fields}` | 構造体フィールドの列 |

### 実装

placeholder の展開は `compiler/synthesis/src/template_engine.rs` が担当します。inverse 側の照合は `compiler/inverse/src/ast_inverse/engine.rs` が担い、どちらも同じ template を解釈する一方で、走査方向と出力の役割は対称になっています。

## 2. `«directive»` directive

`«...»` は Guillemet (二重山括弧) で囲んだ semantic marker です。forward では出力から除去され、inverse では block 境界の検出などに利用されます。

### 現状の directive

現在サポートされているのは 1 種類のみです。

| directive | 所属 | 役割 |
|---|---|---|
| `«dedent»` | python の product template の `close` | indent が減少した位置を block 終端とみなす |

python の mapping.lex では、以下の形で使われています (`axiom/target/lang/python/mapping.lex:15-19`)。

```kdl
product {
  emit {
    visibility ""
    keyword "class"
    open "(TypedDict):"
    close "«dedent»"
    field-format "    {name}: {type}"
    empty-body "    pass"
  }
}
```

python は brace 言語と異なり block 終端を記号で示さないため、列 0 に非空白行が現れた位置を block 終端とみなす必要があります。`close "«dedent»"` はこの挙動を inverse に伝えるための指示です。

### forward 側での除去

forward 出力からは directive を除去します。具体的には、`compiler/synthesis/src/template_engine.rs` の `strip_invert_directives` が identifier 形 (英数字・`_`・`-` のみからなる内容) を `«»` で囲んだ marker を剥がし、それ以外の内容を含む Guillemet (例えば source リテラルに含まれる `«任意文字列»`) は保持されます。

### inverse 側での解釈

inverse 側では `compiler/inverse/src/ast_inverse/engine.rs` の `find_matching_close_directive` が directive 名を見て分岐します。現状は `dedent` のみが実装されており、該当時は `find_dedent_boundary` が column 0 の非空白行を探して block 終端を判定します。

### 将来の directive 追加

新しい directive を加える場合、以下の 2 箇所に手を入れます。

1. 対象言語の mapping.lex の該当 template に `«新ディレクティブ»` を置く
2. `find_matching_close_directive` の分岐表に新しい処理を追加する

forward 側の `strip_invert_directives` は identifier 形を汎用的に除去するため、directive 名が英数字・`_`・`-` のみで構成されていれば追加変更は不要です。

## 3. forward と inverse の対応

同じ template が両方向で機能する仕組みは、`pattern { derive-from "emit" }` 宣言で表現します。

```kdl
binding { emit "{name}" pattern { derive-from "emit" } }
```

この `binding` は rust の mapping.lex の pattern セクションで使われている定義です (`axiom/target/lang/rust/mapping.lex:93`)。`pattern` セクションは inverse 側の照合定義で、`derive-from "emit"` は「照合パターンは emit template から派生する」という指示です。全 21 言語の mapping.lex に共通して現れます。

利点は 3 点です。

1. 同じ構文を forward と inverse で二重に書く必要がない
2. emit を変更すると pattern も自動追従する
3. forward / inverse の不整合が定義時点で排除される

`«directive»` は forward 出力から除去される一方で、派生した pattern 側には残り続けます。emit と pattern は同じ template から派生する対であり、directive は inverse 側だけに残ります。

## 4. 具体例の並置

2 系統の記法は、言語の block 構造の違いに対応します。

### rust (brace 系、directive 不要)

```kdl
binding { emit "let {name} = {value};" }
```

brace 言語では block 境界が `{` `}` や `;` で構文的に明示されるため、`{name}` / `{value}` の placeholder のみで forward / inverse の双方が閉じます。

### python (indent 系、directive 必要)

```kdl
product {
  emit {
    keyword "class"
    open "(TypedDict):"
    close "«dedent»"
    field-format "    {name}: {type}"
  }
}
```

indent 言語では block 終端を記号で示さないため、`close "«dedent»"` が inverse に終端検出方法を指示します。forward 出力には `«dedent»` は現れません (`strip_invert_directives` で除去される)。

## 5. 関連 docs

- IR 階層と inverse パイプライン: [../architecture/ir.md](../architecture/ir.md)
- Lex 層の分類: [layers.md](layers.md)
- forward emit の実装: [../architecture/synthesis.md](../architecture/synthesis.md)
- 対象言語の一覧: [target-languages.md](target-languages.md)
