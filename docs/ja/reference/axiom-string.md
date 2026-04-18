# axiom: string / bytes

所属 cratis: `str`, `bytes`

文字列とバイト列の axiom。`axiom/str`, `axiom/bytes` を扱います。

## str (`axiom/str/`)

### `ops.lex`: 基本操作

| 宣言 | 入力 | 出力 |
|---|---|---|
| `str.len` | s | length |
| `str.char_len` | s | length |
| `str.concat` | (a, b) | result |
| `str.slice` | (s, start, end) | slice |
| `str.find` | (haystack, needle) | index |
| `str.eq` | (a, b) | result |
| `str.compare` | (a, b) | ordering |
| `str.get_char` | (s, i) | ch |
| `str.from_int` | n | s |
| `str.parse_int` | s | n |

### `diff.lex`: 差分

| 宣言 | 用途 |
|---|---|
| `str.diff.diff` | 2 文字列の差分 |
| `str.diff.diff_intra_line` | 行内差分 |
| `str.diff.to_hunks` | hunk 形式 |
| `str.diff.to_patches` | patch 形式 |

### `patch.lex`: パッチ

| 宣言 | 用途 |
|---|---|
| `str.patch.new` | 空 patch |
| `str.patch.insert`, `delete`, `replacement` | 編集操作 |
| `str.patch.start`, `end` | 範囲 |
| `str.patch.apply`, `apply_batch` | 適用 |
| `str.patch.validate` | 検証 |
| `str.patch.inverse` | 逆変換 |

### `fuzzy.lex`: fuzzy 検索

| 宣言 | 用途 |
|---|---|
| `str.fuzzy.score` | fuzzy スコア |
| `str.fuzzy.match_indices` | マッチ位置 |

### `path.lex`: パス操作

| 宣言 | 用途 |
|---|---|
| `str.path.parent` | 親パス |
| `str.path.join` | 結合 |
| `str.path.contains` | 包含判定 |

### `wrap.lex`: 折り返し

| 宣言 | 用途 |
|---|---|
| `str.wrap.wrap_line` | 折り返し |
| `str.wrap.text_width` | 表示幅 |

### `derives.lex`

derives 宣言 (vectorize / lift 等) を集約。

## bytes (`axiom/bytes/ops.lex`)

| 宣言 | 入力 | 出力 |
|---|---|---|
| `bytes.len` | b | length |
| `bytes.get` | (b, i) | byte |
| `bytes.slice` | (b, start, end) | slice |
| `bytes.concat` | (a, b) | result |
| `bytes.eq` | (a, b) | result |

## 依存関係

- `str` の実装は [neco-textpatch](https://github.com/barineco/neco-crates/tree/main/neco-textpatch) (patch / diff), [neco-fuzzy](https://github.com/barineco/neco-crates/tree/main/neco-fuzzy) (fuzzy) 等に依存 (morph.lex 経由で各言語で解決)
- `bytes` は `crypto`, `json`, `cbor`, `cid`, `car` から参照される基盤
- 両者とも laplan の基盤 axiom で、AT Protocol 固有ではない
- 言語別の実装差し替えについては [axiom-bindings](../guide/axiom-bindings.md) を参照
