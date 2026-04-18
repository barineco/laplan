# axiom: memory

所属 cratis: `memory`

メモリモデルの axiom。`axiom/memory/` に配置されます。WASM のリニアメモリ上の load/store を Lex₀ 射として表現します。

## linear (`axiom/memory/linear.lex`)

WASM 32-bit アドレス空間 (`ptr: i32`) への load / store プリミティブ。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `memory.linear.load_i32` | ptr | result (i32) |
| `memory.linear.store_i32` | (ptr, value) | ∅ |
| `memory.linear.load_i64` | ptr | result (i64) |
| `memory.linear.store_i64` | (ptr, value) | ∅ |
| (同様に f32 / f64 系) | | |

## access (`axiom/memory/access.lex`)

データ構造アクセスパターン。load / store の NSID が nsid-map 経由で WASM opcode に解決されます。

```kdl
derives "array.len" via memory_access {
    op "load"
    opcode "i32.load"
    offset "ptr"
}

derives "array.get_f64" via memory_access {
    op "load"
    opcode "f64.load"
    offset "i32.add(i32.add(ptr, 4), i32.mul(index, 8))"
}
```

典型パターン:

| 対象 | レイアウト |
|---|---|
| `array` | 先頭 4 bytes が length、以降が要素列 |
| `algebra.matrix` | row-major f64 配列 |

## 層分類

| 層 | 対象 |
|---|---|
| Lex₀ | load / store の素射 |
| Lex₂ (derives) | `memory_access` による opcode 解決 |

## 依存関係

- `i32`, `i64`, `f32`, `f64` axiom ([axiom-numeric](axiom-numeric.md))
- WASM バイナリ emit ([architecture/compiler.md](../architecture/compiler.md)) と協調
- opcode mapping は `ir::mapping::parse_wasm_mapping_kdl` が読む
