# axiom: serialization

Owning cratis: `json`, `cbor`, `kdl`

Axiom for serialization. Covers `axiom/json`, `axiom/cbor`, and `axiom/kdl`.

## json (`axiom/json/ops.lex`)

| Declaration | Input | Output |
|---|---|---|
| `json.encode` | value | bytes / string |
| `json.parse` | bytes / string | value |

## cbor (`axiom/cbor/ops.lex`)

| Declaration | Input | Output |
|---|---|---|
| `cbor.encode` | value | bytes |
| `cbor.decode` | bytes | value |
| `cbor.encode_dag` | dag | bytes |
| `cbor.decode_dag` | bytes | dag |

Variants with `_dag` are for IPLD DAG-CBOR (with ordering and deterministic encoding). Used in combination with content addressing.

## kdl (`axiom/kdl/ops.lex`)

| Declaration | Input | Output |
|---|---|---|
| `kdl.parse` | text | doc |
| `kdl.serialize` | doc | text |

## Layer Classification

| Layer | Target |
|---|---|
| Lex₀ | Morphisms to `bytes`, `string` |
| Lex₁ | Morphisms for each encode / parse |
| Lex₂ | Symmetry such as `json <-> cbor` described with law (can be added to the relevant module) |

## Dependencies

- [`neco-json`](https://github.com/barineco/neco-crates/tree/main/neco-json), [`neco-cbor`](https://github.com/barineco/neco-crates/tree/main/neco-cbor), [`neco-kdl`](https://github.com/barineco/neco-crates/tree/main/neco-kdl) (language implementations specified in morph.lex).
- Depends on the `bytes` axiom ([axiom-string](axiom-string.md)).
- Used by `cid` and `car` via DAG-CBOR ([axiom-content-addressing](axiom-content-addressing.md)).
