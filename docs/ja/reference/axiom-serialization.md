# axiom: serialization

所属 cratis: `json`, `cbor`, `kdl`

シリアライゼーションの axiom。`axiom/json`, `axiom/cbor`, `axiom/kdl` を扱います。

## json (`axiom/json/ops.lex`)

| 宣言 | 入力 | 出力 |
|---|---|---|
| `json.encode` | value | bytes / string |
| `json.parse` | bytes / string | value |

## cbor (`axiom/cbor/ops.lex`)

| 宣言 | 入力 | 出力 |
|---|---|---|
| `cbor.encode` | value | bytes |
| `cbor.decode` | bytes | value |
| `cbor.encode_dag` | dag | bytes |
| `cbor.decode_dag` | bytes | dag |

`_dag` 付きは IPLD DAG-CBOR (順序付け・deterministic 化) の variant です。content-addressing と組み合わせて使います。

## kdl (`axiom/kdl/ops.lex`)

| 宣言 | 入力 | 出力 |
|---|---|---|
| `kdl.parse` | text | doc |
| `kdl.serialize` | doc | text |

## 層分類

| 層 | 対象 |
|---|---|
| Lex₀ | `bytes`, `string` への射 |
| Lex₁ | 各 encode / parse の morphism |
| Lex₂ | `json <-> cbor` 等の対称性を law で記述 (該当モジュールに追加可能) |

## 依存関係

- [neco-json](https://github.com/barineco/neco-crates/tree/main/neco-json), [neco-cbor](https://github.com/barineco/neco-crates/tree/main/neco-cbor), [neco-kdl](https://github.com/barineco/neco-crates/tree/main/neco-kdl) (morph.lex で各言語実装を指定)
- `bytes` axiom に依存 ([axiom-string](axiom-string.md))
- `cid`, `car` から DAG-CBOR 経由で利用される ([axiom-content-addressing](axiom-content-addressing.md))
- 言語別の実装差し替えについては [axiom-bindings](../guide/axiom-bindings.md) を参照
