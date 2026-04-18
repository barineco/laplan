# axiom: numeric

Owning cratis: `i32`, `i64`, `f32`, `f64`, `bool`

Axiom for numeric primitives. Covers `axiom/i32`, `axiom/i64`, `axiom/f32`, `axiom/f64`, and `axiom/bool`. All are pure functions at Lex₀ + Lex₁, with morphisms that are always fireable: `requires = ∅`, `consumes = ∅`.

## i32 / i64 (Integer)

### Arithmetic (`axiom/i32/arith.lex`, `axiom/i64/arith.lex`)

| Declaration | i32 | i64 | Input | Output |
|---|---|---|---|---|
| `add` | ✓ | ✓ | (a, b) | result |
| `sub` / `subtract` | ✓ | ✓ | (a, b) | result |
| `multiply` | ✓ | ✓ | (a, b) | result |
| `negate` | ✓ | ✓ | a | result |
| `abs` | ✓ | ✓ | a | result |
| `divmod` | ✓ | ✓ | (a, b) | (q, r) |
| `compare` | ✓ | ✓ | (a, b) | ordering |

### Bitwise Operations (`axiom/i32/bit.lex`, `axiom/i64/bit.lex`)

| Declaration | i32 | i64 |
|---|---|---|
| `and`, `or`, `xor` | ✓ | ✓ |
| `shl`, `shr` | ✓ | ✓ |
| `rotl`, `rotr` | ✓ | ✗ |
| `clz`, `ctz`, `popcnt` | ✓ | ✓ |

## f32 / f64 (Floating Point)

### `axiom/f32/ops.lex`, `axiom/f64/ops.lex`

| Declaration | f32 | f64 |
|---|---|---|
| `add`, `sub`, `mul`, `div` | ✓ | ✓ |
| `sqrt`, `abs`, `neg` | ✓ | ✓ |
| `ceil`, `floor` | ✗ | ✓ |
| `eq`, `lt`, `gt` | ✓ | ✓ |

## bool

### `axiom/bool/ops.lex`

| Declaration | Input | Output |
|---|---|---|
| `bool.and` | (a, b) | result |
| `bool.or` | (a, b) | result |
| `bool.not` | a | result |
| `bool.xor` | (a, b) | result |

## Layer Classification

| Layer | Target |
|---|---|
| Lex₀ (type) | `type i32`, `type i64`, `type f32`, `type f64`, `type bool` |
| Lex₀ (lexicon) | `lex i32.add { procedure { input; output } }`, etc. |
| Lex₁ (morphism) | Morphisms for each operation (searched by the solver) |
| Lex₂ (family) | SIMD / product family derivation via [axiom-algebra](axiom-algebra.md) |

## Dependencies

- These are the lowest-level axioms in laplan and depend on no other axioms.
- The higher-level `axiom/algebra/` derives Vec2 / Complex, etc. via family (vectorize, product).
- `axiom/crypto/` uses these via `bytes`.
