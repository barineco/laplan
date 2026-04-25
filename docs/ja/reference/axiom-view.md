# axiom/view リファレンス

`axiom/view/` は Petri net グラフ UI が使う Lex₀ 型宣言をまとめた cratis です。cratis 名は `"view"`、所属 package は `view`。`compiler/ir` の `meta` パース層と組み合わせて、座標・viewport・経路情報を `.lex` 本体とは分離して管理します。

## cratis 構成

```
axiom/view/
  cratis.lex               cratis "view" (provides Position, Size, view.graph サブ cratis)
  position.lex             view.Position / view.Size
  graph/
    cratis.lex             cratis "view.graph"
    node-box.lex           view.graph.NodeBox
    edge-route.lex         view.graph.EdgeRoute
    viewport.lex           view.graph.ViewportTransform
```

## 汎用型 (view cratis)

### view.Position

2D 座標。world 座標系で使う。

| フィールド | 型 | 説明 |
|---|---|---|
| `x` | `f64` | 横方向 |
| `y` | `f64` | 縦方向 |

### view.Size

矩形サイズ。

| フィールド | 型 | 説明 |
|---|---|---|
| `w` | `f64` | 幅 |
| `h` | `f64` | 高さ |

## グラフ特化型 (view.graph cratis)

### view.graph.NodeBox

ノードの位置とサイズ。

| フィールド | 型 | 説明 |
|---|---|---|
| `pos` | `view.Position` | 左上座標 |
| `size` | `view.Size` | ノード矩形サイズ |

### view.graph.EdgeRoute

エッジの経路情報。制御点列を持つ。

| フィールド | 型 | 説明 |
|---|---|---|
| `from` | `view.Position` | 始点 |
| `to` | `view.Position` | 終点 |
| `control` | `[view.Position]` | 制御点列 (bezier / spline の中間点) |
| `style` | `str` | 描画スタイル (`bezier` / `orthogonal` 等) |

### view.graph.ViewportTransform

ビューポートの状態。

| フィールド | 型 | 説明 |
|---|---|---|
| `center_x` | `f64` | ビューポート中心の world x 座標 |
| `center_y` | `f64` | ビューポート中心の world y 座標 |
| `view_size` | `f64` | ビューポートの world 座標上の幅 |

## Lex₃ meta との関係

`axiom/view/` は Lex₀ の型宣言です。実際のノード座標とビューポート状態は Lex₃ の `meta view.graph.<nsid>` ノードとして保存します。型宣言と値の分離は `.lex` 本体 (ロジック) と UI メタ情報の分離に対応します。

```
Lex₀: axiom/view/graph/node-box.lex    → view.graph.NodeBox の型定義
Lex₃: view/graph/car.parse_v1.lex      → meta "view.graph.car.parse_v1" { ... }
```

`meta` のパースは `compiler/ir` の `parse_meta_kdl()` (src/meta.rs) が担当します。

## 関連

- [reference/layers.md](layers.md) の「meta (Lex₃)」節: meta トップノードの位置づけ
- [guide/wasm-extension.md](../guide/wasm-extension.md): UI パッケージ構成と meta 永続化の流れ
