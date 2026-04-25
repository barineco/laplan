# axiom: string / bytes

Owning cratis: `str`, `bytes`

Axiom for strings and byte sequences. Covers `axiom/str` and `axiom/bytes`.

## str (`axiom/str/`)

### `ops.lex`: Basic Operations

| Declaration | Input | Output |
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

### `diff.lex`: Diff

| Declaration | Purpose |
|---|---|
| `str.diff.diff` | Diff between two strings |
| `str.diff.diff_intra_line` | Intra-line diff |
| `str.diff.to_hunks` | Hunk format |
| `str.diff.to_patches` | Patch format |

### `patch.lex`: Patches

| Declaration | Purpose |
|---|---|
| `str.patch.new` | Empty patch |
| `str.patch.insert`, `delete`, `replacement` | Edit operations |
| `str.patch.start`, `end` | Range |
| `str.patch.apply`, `apply_batch` | Application |
| `str.patch.validate` | Validation |
| `str.patch.inverse` | Inverse transformation |

### `fuzzy.lex`: Fuzzy Search

| Declaration | Purpose |
|---|---|
| `str.fuzzy.score` | Fuzzy score |
| `str.fuzzy.match_indices` | Match positions |

### `path.lex`: Path Operations

| Declaration | Purpose |
|---|---|
| `str.path.parent` | Parent path |
| `str.path.join` | Join paths |
| `str.path.contains` | Containment check |

### `wrap.lex`: Line Wrapping

| Declaration | Purpose |
|---|---|
| `str.wrap.wrap_line` | Line wrapping |
| `str.wrap.text_width` | Display width |

### `derives.lex`

Aggregates derives declarations (vectorize / lift, etc.).

## bytes (`axiom/bytes/ops.lex`)

| Declaration | Input | Output |
|---|---|---|
| `bytes.len` | b | length |
| `bytes.get` | (b, i) | byte |
| `bytes.slice` | (b, start, end) | slice |
| `bytes.concat` | (a, b) | result |
| `bytes.eq` | (a, b) | result |

## Dependencies

- `str` implementations depend on [`neco-textpatch`](https://github.com/barineco/neco-crates/tree/main/neco-textpatch) (patch / diff), [`neco-fuzzy`](https://github.com/barineco/neco-crates/tree/main/neco-fuzzy) (fuzzy), etc. (resolved per language via morph.lex).
- `bytes` is the foundation referenced by `crypto`, `json`, `cbor`, `cid`, and `car`.
- Both are foundational axioms in laplan and are not AT Protocol-specific.
