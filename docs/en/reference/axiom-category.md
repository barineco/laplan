# axiom: category

Owning cratis: `category`

Category theory primitives. Placed under `axiom/category/`. Serves as the basis for solver pruning, identification, and derivation as Lex₂ structural constraints.

## compose (`compose.lex`)

Derives a new API by composing multiple existing APIs.

```kdl
derives "math.composed.sum_of_squares" via compose {
    sources "i32.multiply" "i32.add"
    // ...
}
```

- `sources` lists the source APIs as peers.
- Reassignment to the same-named output is allowed (first occurrence = Let, subsequent = Assign).

## dual (`dual.lex`)

Declares commutativity conditions for forward / reverse queries on the same record type.

```kdl
dual follow_dual {
    // forward:  getFollows(A)   ∋ B
    // reverse:  getFollowers(B) ∋ A
}
```

The solver can use the symmetry of bidirectional queries.

## fold (`fold.lex`)

Declares the invariant that accumulated records match the count field of a view.

```kdl
invariant like_count {
    record "<record-nsid>"
    // count(record where target = subject) == view.count-field
}
```

## law (`law.lex`)

Algebraic law declarations. Provides mathematical backing for existing patterns (fold, inverse, dual) and serves as solver pruning conditions. The place to describe group axioms (commutativity, associativity, identity, inverse).

```kdl
law arith_comm {
    // add(a, b) == add(b, a)
}
```

## lift (`lift.lex`)

Automatically derives a morphism `batch(f): [A] → [B]` from a morphism `f: A → B`. Uses `derives ... via batch`:

```kdl
derives "X.getProfiles" from "X.getProfile" via batch { max-length 25 }
```

## product (`product.lex`)

Derives SIMD transitions from scalar members of a family. `vectorize` cooperates with axiom/algebra/family.

```kdl
derives "simd.f64x2" from family="Numeric" member="number" via vectorize {
    lane-width 2
    target "wasm"
}
```

## restrict (`restrict.lex`)

Provides specialize, which binds parameters to obtain a new morphism.

```kdl
derives "algebra.modular.mod_inv" from "algebra.modular.mod_pow" via specialize {
    bind exp="i64.sub(p, 2)"
}
```

## Layer Classification

| Layer | Target |
|---|---|
| Lex₂ (constrain) | All declarations. Not searched by the solver; used as decision material |

## Dependencies

- Each derivation references concrete morphisms from axiom/i32, axiom/algebra, etc.
- compose / lift / specialize / vectorize are expanded into concrete transitions by `derives_resolve.rs` in synthesis ([architecture/synthesis.md](../architecture/synthesis.md)).
