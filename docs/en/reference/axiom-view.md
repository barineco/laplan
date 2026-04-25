# axiom/view Reference

`axiom/view/` is a cratis that holds Lex₀ type declarations used by the Petri net graph UI. The cratis name is `"view"` and the package is `view`. Combined with the `meta` parse layer in `compiler/ir`, it manages node coordinates, viewport state, and edge routing data separately from the body `.lex`.

## Cratis Structure

```
axiom/view/
  cratis.lex               cratis "view" (provides Position, Size, view.graph sub-cratis)
  position.lex             view.Position / view.Size
  graph/
    cratis.lex             cratis "view.graph"
    node-box.lex           view.graph.NodeBox
    edge-route.lex         view.graph.EdgeRoute
    viewport.lex           view.graph.ViewportTransform
```

## General Types (view cratis)

### view.Position

2D coordinate in world space.

| Field | Type | Description |
|---|---|---|
| `x` | `f64` | Horizontal axis |
| `y` | `f64` | Vertical axis |

### view.Size

Rectangle dimensions.

| Field | Type | Description |
|---|---|---|
| `w` | `f64` | Width |
| `h` | `f64` | Height |

## Graph-Specific Types (view.graph cratis)

### view.graph.NodeBox

Position and size of a node.

| Field | Type | Description |
|---|---|---|
| `pos` | `view.Position` | Top-left coordinate |
| `size` | `view.Size` | Node rectangle size |

### view.graph.EdgeRoute

Edge routing data with control points.

| Field | Type | Description |
|---|---|---|
| `from` | `view.Position` | Start point |
| `to` | `view.Position` | End point |
| `control` | `[view.Position]` | Control points (intermediate points for bezier / spline) |
| `style` | `str` | Render style (`bezier` / `orthogonal`, etc.) |

### view.graph.ViewportTransform

Viewport state.

| Field | Type | Description |
|---|---|---|
| `center_x` | `f64` | World x coordinate of the viewport center |
| `center_y` | `f64` | World y coordinate of the viewport center |
| `view_size` | `f64` | Width of the viewport in world coordinates |

## Relationship with Lex₃ meta

`axiom/view/` contains Lex₀ type declarations. Actual node coordinates and viewport state are saved as `meta view.graph.<nsid>` nodes at Lex₃. The separation of type declarations from values corresponds to the separation of body `.lex` (logic) from UI metadata.

```
Lex₀: axiom/view/graph/node-box.lex    → type definition for view.graph.NodeBox
Lex₃: view/graph/car.parse_v1.lex      → meta "view.graph.car.parse_v1" { ... }
```

Parsing is handled by `parse_meta_kdl()` in `compiler/ir` (src/meta.rs).

## Related

- [reference/layers.md](layers.md) "meta (Lex₃)" section: position of the meta top node
- [guide/wasm-extension.md](../guide/wasm-extension.md): UI package structure and meta persistence
