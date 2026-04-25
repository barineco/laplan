# axiom: crypto

所属 cratis: `crypto`

暗号プリミティブの axiom。`axiom/crypto/` に self-describing cratis として配置されます。

```
axiom/crypto/
├── cratis.lex
├── hash.lex
├── jwt.lex
├── password.lex
├── secp.lex
└── vault.lex
```

消費者は workspace lib.lex の face で `axiom "crypto"` として参照し、`from="axiom/crypto"` (Builtin source) で解決します。

## hash (`axiom/crypto/hash.lex`)

| 宣言 | 入力 | 出力 |
|---|---|---|
| `crypto.hash.sha256` | data | digest |
| `crypto.hash.sha1` | data | digest |
| `crypto.hash.hmac_sha256` | (key, data) | mac |

実装は [neco-sha2](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [neco-sha1](https://github.com/barineco/neco-crates/tree/main/neco-sha1) 等に対応します。

## secp (`axiom/crypto/secp.lex`)

楕円曲線 (secp256k1 / P-256) 系。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `crypto.secp.sign` | (privkey, msg) | signature |
| `crypto.secp.verify` | (pubkey, msg, sig) | valid |

実装は [neco-secp](https://github.com/barineco/neco-crates/tree/main/neco-secp), [neco-p256](https://github.com/barineco/neco-crates/tree/main/neco-p256) ([neco-galois](https://github.com/barineco/neco-crates/tree/main/neco-galois) 経由の GF(p) 演算) に対応。

## jwt (`axiom/crypto/jwt.lex`)

JSON Web Token (RFC 7519)。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `crypto.jwt.issue` | (subject, secret, expires-in) | token |
| `crypto.jwt.verify` | (token, secret) | (subject, issued-at, expires-at) |

TTL 付き capability として solver が扱い、`TimedCapabilityNoRenewal` 診断と連携します。

## password (`axiom/crypto/password.lex`)

パスワードハッシュ。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `crypto.password.hash` | password | password-hash |
| `crypto.password.verify` | (password, password-hash) | valid |

Argon2id 等の memory-hard アルゴリズムを想定 (morph.lex で各言語実装を指定)。

## vault (`axiom/crypto/vault.lex`)

鍵管理。状態を持つ vault ハンドルは Lex₀ 外に出し、鍵素材を bytes として境界を越えさせます。

| 宣言 | 入力 | 出力 |
|---|---|---|
| `crypto.vault.create_keypair` | ∅ | keypair |
| `crypto.vault.export_did_key` | pubkey | did_key |

`did:key` は W3C DID method の汎用表現です。

## 層分類

| 層 | 対象 |
|---|---|
| Lex₀ | `bytes`, `string` の参照 |
| Lex₁ | sign / verify / hash の morphism |
| Lex₂ | TTL 付き capability、署名検証の invariant |

## 依存関係

- `bytes` axiom ([axiom-string](axiom-string.md))
- 実装は [neco-sha2](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [neco-sha1](https://github.com/barineco/neco-crates/tree/main/neco-sha1), [neco-secp](https://github.com/barineco/neco-crates/tree/main/neco-secp), [neco-p256](https://github.com/barineco/neco-crates/tree/main/neco-p256), [neco-galois](https://github.com/barineco/neco-crates/tree/main/neco-galois), [neco-vault](https://github.com/barineco/neco-crates/tree/main/neco-vault) 等 (morph.lex で各言語解決)
- AT Protocol 専用ではなく汎用の暗号プリミティブ
- 言語別の実装差し替えについては [axiom-bindings](../guide/axiom-bindings.md) を参照
