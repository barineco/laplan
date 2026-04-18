# axiom: category

所属 cratis: `category`

圏論プリミティブ。`axiom/category/` に配置されます。Lex₂ の構造制約として solver の枝刈り・同一視・導出の根拠になります。

## compose (`compose.lex`)

複数の既存 API を合成して新 API を導出します。

```kdl
derives "math.composed.sum_of_squares" via compose {
    sources "i32.multiply" "i32.add"
    // ...
}
```

- `sources` で合成元 API を対等に列挙
- 同名 output への再代入は許容 (初回 = Let, 2 回目以降 = Assign)

## dual (`dual.lex`)

同一 record type に対する forward / reverse クエリの可換条件を宣言します。

```kdl
dual follow_dual {
    // forward:  getFollows(A)   ∋ B
    // reverse:  getFollowers(B) ∋ A
}
```

双方向クエリの対称性を solver が利用できます。

## fold (`fold.lex`)

record の蓄積と view の count フィールドが一致する不変条件を宣言します。

```kdl
invariant like_count {
    record "<record-nsid>"
    // count(record where target = subject) == view.count-field
}
```

## law (`law.lex`)

代数的法則宣言。既存 pattern (fold, inverse, dual) に数学的裏付けを与え、solver の枝刈り条件として使われます。群公理 (交換律、結合律、単位元、逆元) を記述する場です。

```kdl
law arith_comm {
    // add(a, b) == add(b, a)
}
```

## lift (`lift.lex`)

射 f: A → B から射 batch(f): [A] → [B] を自動導出します。`derives ... via batch`:

```kdl
derives "X.getProfiles" from "X.getProfile" via batch { max-length 25 }
```

## product (`product.lex`)

family の scalar member から SIMD transition を導出します。`vectorize` は axiom/algebra/family と連携します。

```kdl
derives "simd.f64x2" from family="Numeric" member="number" via vectorize {
    lane-width 2
    target "wasm"
}
```

## restrict (`restrict.lex`)

パラメータを束縛して新しい射を得る specialize を提供します。

```kdl
derives "algebra.modular.mod_inv" from "algebra.modular.mod_pow" via specialize {
    bind exp="i64.sub(p, 2)"
}
```

## 層分類

| 層 | 対象 |
|---|---|
| Lex₂ (constrain) | 全 declaration。solver は探索せず、判断材料として使う |

## 依存関係

- 各導出は axiom/i32, axiom/algebra 等の具体射を参照
- compose / lift / specialize / vectorize は synthesis の `derives_resolve.rs` で具体的 transition に展開される ([architecture/synthesis.md](../architecture/synthesis.md))
