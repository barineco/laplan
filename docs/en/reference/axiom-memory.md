# axiom: memory

Owning cratis: `memory`

Axiom for the memory model. Placed under `axiom/memory/`. Expresses load/store operations on WASM linear memory as Lex₀ morphisms.

## linear (`axiom/memory/linear.lex`)

Load / store primitives for the WASM 32-bit address space (`ptr: i32`).

| Declaration | Input | Output |
|---|---|---|
| `memory.linear.load_i32` | ptr | result (i32) |
| `memory.linear.store_i32` | (ptr, value) | ∅ |
| `memory.linear.load_i64` | ptr | result (i64) |
| `memory.linear.store_i64` | (ptr, value) | ∅ |
| (similarly for f32 / f64) | | |

## access (`axiom/memory/access.lex`)

Data structure access patterns. load / store NSIDs are resolved to WASM opcodes via the nsid-map.

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

Typical patterns:

| Target | Layout |
|---|---|
| `array` | First 4 bytes are length, followed by element sequence |
| `algebra.matrix` | Row-major f64 array |

## Layer Classification

| Layer | Target |
|---|---|
| Lex₀ | Primitive load / store morphisms |
| Lex₂ (derives) | Opcode resolution via `memory_access` |

## Dependencies

- `i32`, `i64`, `f32`, `f64` axioms ([axiom-numeric](axiom-numeric.md)).
- Cooperates with WASM binary emit ([architecture/compiler.md](../architecture/compiler.md)).
- The opcode mapping is read by `ir::mapping::parse_wasm_mapping_kdl`.
