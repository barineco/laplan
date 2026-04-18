# axiom: algebra

Owning cratis: `algebra`

Axiom for algebraic structures and families. Covers `axiom/algebra/`.

## family (`axiom/algebra/family/`)

A `family` is a closed set of types sharing isomorphic operations. SIMD / product / vectorize variants are automatically derived from scalar members.

| File | Declaration | Description |
|---|---|---|
| `numeric.lex` | `family "Numeric"` | Base for scalar numeric types such as i32 / i64 / f32 / f64 |
| `complex.lex` | `family "Complex"` | `product("Numeric", 2)` complex numbers |
| `radian.lex` | `family "Radian"` | Angle type |
| `vec2.lex` | `family "Vec2"` | `product("Numeric", 2)` vector |
| `vec3.lex` | `family "Vec3"` | `product("Numeric", 3)` vector |

### family Syntax

```kdl
family "Complex" = product("Numeric", 2) {
    signature "mul" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "abs" { in { (Self)z } out { (f64)result } }
    signature "conjugate" { in { (Self)z } out { (Self)result } }
}
```

- `product("Base", N)`: composite type of N copies of Base.
- `signature`: declaration of type-specific operations. Component-wise operations are derived automatically.
- Connection to vectorize (SIMD) is available.

## Algebraic Structures

| Directory | File | Declaration |
|---|---|---|
| `field/` | `gf256.lex` | GF(2⁸) finite field arithmetic |
| `group/` | `inverse.lex` | Group inverse element |
| `modular/` | `ops.lex` | Modular arithmetic |
| `monoid/` | `power.lex` | Monoid exponentiation |
| `matrix/` | `ops.lex` | Matrix operations |

## recursive

`axiom/algebra/recursive/` is a collection of safe recursive patterns (Lex₁ layer, integrable into solver paths).

| Directory | Pattern |
|---|---|
| `gcd/` | Greatest common divisor |
| `mod-inverse/` | Modular inverse (extended Euclidean) |
| `power/` | Exponentiation |

### Lex₁ safe recursive placement

11 patterns of `recursive.bounded` / `recursive.decreasing` declarations searchable as solver paths. By category:

| Category | Patterns |
|---|---|
| algebra | gcd, power, mod-inverse |
| search | bisect, binary-search, ternary-search |
| solve | newton, fixed-point, bisect-root |
| divide | merge-sort, quickselect |

Lex₂-layer unsafe recursive (via recursive) is label-only and outside the solver scope.

## Dependencies

- Depends on `i32` / `i64` / `f32` / `f64` axioms as the base ([axiom-numeric](axiom-numeric.md)).
- Connects to GF(p) arithmetic in [`neco-galois`](https://github.com/barineco/neco-crates/tree/main/neco-galois) (implementations via morph.lex).
- product / vectorize derivation cooperates with `product.lex` in [axiom-category](axiom-category.md).
