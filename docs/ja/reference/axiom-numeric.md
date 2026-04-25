# axiom: numeric

所属 cratis: `i32`, `i64`, `f32`, `f64`, `bool`

数値プリミティブの axiom。`axiom/i32`, `axiom/i64`, `axiom/f32`, `axiom/f64`, `axiom/bool` を扱います。全て Lex₀ + Lex₁ の純粋関数で、morphism は `requires = ∅`, `consumes = ∅` で常に発火可能です。

## i32 / i64 (整数)

### 算術 (`axiom/i32/arith.lex`, `axiom/i64/arith.lex`)

| 宣言 | i32 | i64 | 入力 | 出力 |
|---|---|---|---|---|
| `add` | ✓ | ✓ | (a, b) | result |
| `sub` / `subtract` | ✓ | ✓ | (a, b) | result |
| `multiply` | ✓ | ✓ | (a, b) | result |
| `negate` | ✓ | ✓ | a | result |
| `abs` | ✓ | ✓ | a | result |
| `divmod` | ✓ | ✓ | (a, b) | (q, r) |
| `compare` | ✓ | ✓ | (a, b) | ordering |

### ビット演算 (`axiom/i32/bit.lex`, `axiom/i64/bit.lex`)

| 宣言 | i32 | i64 |
|---|---|---|
| `and`, `or`, `xor` | ✓ | ✓ |
| `shl`, `shr` | ✓ | ✓ |
| `rotl`, `rotr` | ✓ | ✗ |
| `clz`, `ctz`, `popcnt` | ✓ | ✓ |

## f32 / f64 (浮動小数点)

### `axiom/f32/ops.lex`, `axiom/f64/ops.lex`

| 宣言 | f32 | f64 |
|---|---|---|
| `add`, `sub`, `mul`, `div` | ✓ | ✓ |
| `sqrt`, `abs`, `neg` | ✓ | ✓ |
| `ceil`, `floor` | ✗ | ✓ |
| `eq`, `lt`, `gt` | ✓ | ✓ |

## bool

### `axiom/bool/ops.lex`

| 宣言 | 入力 | 出力 |
|---|---|---|
| `bool.and` | (a, b) | result |
| `bool.or` | (a, b) | result |
| `bool.not` | a | result |
| `bool.xor` | (a, b) | result |

## 層分類

| 層 | 対象 |
|---|---|
| Lex₀ (type) | `type i32`, `type i64`, `type f32`, `type f64`, `type bool` |
| Lex₀ (lexicon) | `lex i32.add { procedure { input; output } }` 等 |
| Lex₁ (morphism) | 各演算の射 (solver が探索) |
| Lex₂ (family) | [axiom-algebra](axiom-algebra.md) で SIMD / product family を導出 |

## 依存関係

- これらは laplan の最下層 axiom で、他 axiom に依存しません
- 上位の `axiom/algebra/` が family (vectorize, product) を介して Vec2 / Complex 等を導出
- `axiom/crypto/` が `bytes` 経由で利用
