# axiom: content addressing

所属 cratis: `cid`, `car`

コンテンツアドレッシングの axiom。`axiom/cid`, `axiom/car` を扱います。

## cid (`axiom/cid/ops.lex`)

CID (Content IDentifier): content-addressed なブロックの識別子です。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `cid.compute` | bytes | cid |
| `cid.from_bytes` | bytes | cid |
| `cid.to_bytes` | cid | bytes |
| `cid.to_multibase` | cid | text |
| `cid.from_multibase` | text | cid |

### 典型的な組合せ

```
bytes --(cbor.encode_dag)--> bytes --(cid.compute)--> cid
```

CID を得たら、`json.encode` / `cbor.encode` から直接落とすのではなく、DAG-CBOR 経由で決定化した bytes に対して計算します。

## car (`axiom/car/ops.lex`)

CAR (Content Addressable aRchives): 複数ブロックを 1 ファイルにまとめる v1 フォーマット。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `car.parse_v1` | bytes | blocks |
| `car.write_v1` | (root_cids, blocks) | bytes |

## 層分類

| 層 | 対象 |
|---|---|
| Lex₀ | `type cid`, `type bytes` (参照) |
| Lex₁ | compute / parse / write の morphism |

## 依存関係

- `bytes` axiom に依存 ([axiom-string](axiom-string.md))
- `cbor` axiom と組合せて DAG-CBOR を介した determinism を提供 ([axiom-serialization](axiom-serialization.md))
- 実装は [neco-cid](https://github.com/barineco/neco-crates/tree/main/neco-cid), [neco-car](https://github.com/barineco/neco-crates/tree/main/neco-car) (morph.lex で解決)
- AT Protocol の MST (Merkle Search Tree) / firehose / repository 等のユースケースに対応
