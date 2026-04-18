# axiom: content addressing

Owning cratis: `cid`, `car`

Axiom for content addressing. Covers `axiom/cid` and `axiom/car`.

## cid (`axiom/cid/ops.lex`)

CID (Content IDentifier): an identifier for content-addressed blocks.

| Declaration | Input | Output |
|---|---|---|
| `cid.compute` | bytes | cid |
| `cid.from_bytes` | bytes | cid |
| `cid.to_bytes` | cid | bytes |
| `cid.to_multibase` | cid | text |
| `cid.from_multibase` | text | cid |

### Typical Combination

```
bytes --(cbor.encode_dag)--> bytes --(cid.compute)--> cid
```

To obtain a CID, compute it against bytes that have been made deterministic via DAG-CBOR, rather than dropping directly from `json.encode` / `cbor.encode`.

## car (`axiom/car/ops.lex`)

CAR (Content Addressable aRchives): a v1 format that bundles multiple blocks into a single file.

| Declaration | Input | Output |
|---|---|---|
| `car.parse_v1` | bytes | blocks |
| `car.write_v1` | (root_cids, blocks) | bytes |

## Layer Classification

| Layer | Target |
|---|---|
| Lex₀ | `type cid`, `type bytes` (reference) |
| Lex₁ | Morphisms for compute / parse / write |

## Dependencies

- Depends on the `bytes` axiom ([axiom-string](axiom-string.md)).
- Combined with the `cbor` axiom to provide determinism via DAG-CBOR ([axiom-serialization](axiom-serialization.md)).
- Implementations are [`neco-cid`](https://github.com/barineco/neco-crates/tree/main/neco-cid), [`neco-car`](https://github.com/barineco/neco-crates/tree/main/neco-car) (resolved in morph.lex).
- Covers AT Protocol use cases such as MST (Merkle Search Tree), firehose, and repository.
