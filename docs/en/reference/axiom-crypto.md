# axiom: crypto

Owning cratis: `crypto`

Axiom for cryptographic primitives. Placed under `axiom/crypto/` as a self-describing cratis.

```
axiom/crypto/
├── cratis.lex
├── hash.lex
├── jwt.lex
├── password.lex
├── secp.lex
└── vault.lex
```

Consumers reference it as `axiom "crypto"` via the face in the workspace lib.lex, resolved with `from="axiom/crypto"` (Builtin source).

## hash (`axiom/crypto/hash.lex`)

| Declaration | Input | Output |
|---|---|---|
| `crypto.hash.sha256` | data | digest |
| `crypto.hash.sha1` | data | digest |
| `crypto.hash.hmac_sha256` | (key, data) | mac |

Implementations correspond to [`neco-sha2`](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [`neco-sha1`](https://github.com/barineco/neco-crates/tree/main/neco-sha1), etc.

## secp (`axiom/crypto/secp.lex`)

Elliptic curve (secp256k1 / P-256) operations.

| Declaration | Input | Output |
|---|---|---|
| `crypto.secp.sign` | (privkey, msg) | signature |
| `crypto.secp.verify` | (pubkey, msg, sig) | valid |

Implementations correspond to [`neco-secp`](https://github.com/barineco/neco-crates/tree/main/neco-secp), [`neco-p256`](https://github.com/barineco/neco-crates/tree/main/neco-p256) (GF(p) arithmetic via [`neco-galois`](https://github.com/barineco/neco-crates/tree/main/neco-galois)).

## jwt (`axiom/crypto/jwt.lex`)

JSON Web Token (RFC 7519).

| Declaration | Input | Output |
|---|---|---|
| `crypto.jwt.issue` | (subject, secret, expires-in) | token |
| `crypto.jwt.verify` | (token, secret) | (subject, issued-at, expires-at) |

The solver treats this as a TTL-bearing capability and integrates with the `TimedCapabilityNoRenewal` diagnostic.

## password (`axiom/crypto/password.lex`)

Password hashing.

| Declaration | Input | Output |
|---|---|---|
| `crypto.password.hash` | password | password-hash |
| `crypto.password.verify` | (password, password-hash) | valid |

Assumes memory-hard algorithms such as Argon2id (language implementations specified in morph.lex).

## vault (`axiom/crypto/vault.lex`)

Key management. Stateful vault handles stay outside Lex₀; key material crosses boundaries as `bytes`.

| Declaration | Input | Output |
|---|---|---|
| `crypto.vault.create_keypair` | ∅ | keypair |
| `crypto.vault.export_did_key` | pubkey | did_key |

`did:key` is the universal representation of the W3C DID method.

## Layer Classification

| Layer | Target |
|---|---|
| Lex₀ | References to `bytes`, `string` |
| Lex₁ | Morphisms for sign / verify / hash |
| Lex₂ | TTL-bearing capabilities, invariants for signature verification |

## Dependencies

- `bytes` axiom ([axiom-string](axiom-string.md)).
- Implementations are [`neco-sha2`](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [`neco-sha1`](https://github.com/barineco/neco-crates/tree/main/neco-sha1), [`neco-secp`](https://github.com/barineco/neco-crates/tree/main/neco-secp), [`neco-p256`](https://github.com/barineco/neco-crates/tree/main/neco-p256), [`neco-galois`](https://github.com/barineco/neco-crates/tree/main/neco-galois), [`neco-vault`](https://github.com/barineco/neco-crates/tree/main/neco-vault), etc. (resolved per language in morph.lex).
- General-purpose cryptographic primitives, not AT Protocol-specific.
