# axiom: algebra

所属 cratis: `algebra`

代数構造と family の axiom。`axiom/algebra/` を扱います。

## family (`axiom/algebra/family/`)

`family` は同型演算を共有する閉じた型の集合です。scalar member から SIMD / product / vectorize を自動導出します。

| ファイル | 宣言 | 内容 |
|---|---|---|
| `numeric.lex` | `family "Numeric"` | i32 / i64 / f32 / f64 等のスカラー数値の基底 |
| `complex.lex` | `family "Complex"` | `product("Numeric", 2)` 複素数 |
| `radian.lex` | `family "Radian"` | 角度型 |
| `vec2.lex` | `family "Vec2"` | `product("Numeric", 2)` ベクトル |
| `vec3.lex` | `family "Vec3"` | `product("Numeric", 3)` ベクトル |

### family 構文

```kdl
family "Complex" = product("Numeric", 2) {
    signature "mul" { in { (Self)a; (Self)b } out { (Self)result } }
    signature "abs" { in { (Self)z } out { (f64)result } }
    signature "conjugate" { in { (Self)z } out { (Self)result } }
}
```

- `product("Base", N)`: Base を N 個並べた合成型
- `signature`: 固有演算の宣言。component-wise 演算は自動導出
- vectorize (SIMD) への接続あり

## 代数構造

| ディレクトリ | ファイル | 宣言 |
|---|---|---|
| `field/` | `gf256.lex` | GF(2⁸) 有限体演算 |
| `group/` | `inverse.lex` | 群の逆元 |
| `modular/` | `ops.lex` | 剰余類演算 |
| `monoid/` | `power.lex` | モノイドのべき乗 |
| `matrix/` | `ops.lex` | 行列演算 |

## recursive

`axiom/algebra/recursive/` は safe recursive (Lex₁ 層。solver 経路に組み込み可) のパターン集です。

| ディレクトリ | パターン |
|---|---|
| `gcd/` | 最大公約数 |
| `mod-inverse/` | 剰余逆元 (拡張 Euclid) |
| `power/` | 冪乗 |

### Lex₁ safe recursive の配置状況

solver が経路として探索可能な `recursive.bounded` / `recursive.decreasing` の宣言が 11 パターン配置されています。分野別:

| カテゴリ | パターン |
|---|---|
| algebra | gcd, power, mod-inverse |
| search | bisect, binary-search, ternary-search |
| solve | newton, fixed-point, bisect-root |
| divide | merge-sort, quickselect |

Lex₂ 層の unsafe recursive (via recursive) はラベルのみで solver 範囲外です。

## 依存関係

- 基底として `i32` / `i64` / `f32` / `f64` axiom に依存 ([axiom-numeric](axiom-numeric.md))
- [neco-galois](https://github.com/barineco/neco-crates/tree/main/neco-galois) の GF(p) 演算と接続 (実装は morph.lex 経由)
- product / vectorize の導出は [axiom-category](axiom-category.md) の `product.lex` と連携
