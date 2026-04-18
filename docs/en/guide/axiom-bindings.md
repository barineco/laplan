# Per-Language Implementation Substitution for axioms

axiom declares type and operation interfaces, but implementations differ by language. In Rust, pure Rust implementations from the neco-crates family are used. In Python, the `cryptography` package is used instead; in Go, the standard library `crypto` package is used. Each language uses the implementation that fits its ecosystem.

This substitution is managed declaratively in the `bindings {}` section of `mapping.lex`.

## How It Works

Each language target definition (`axiom/target/lang/{lang}/mapping.lex`) has a `bindings {}` section that maps axiom operation names to language-specific packages and import paths.

```kdl
// axiom/target/lang/python/mapping.lex
bindings {
    "signature.verify.ES256K" package="cryptography" version="41.0" \
        import="cryptography.hazmat.primitives.asymmetric.ec.ECDSA"
    "signature.verify.P-256"  package="cryptography" version="41.0" \
        import="cryptography.hazmat.primitives.asymmetric.ec.ECDSA"
}
```

synthesis reads these declarations and assembles import statements and package dependencies in generated code on a per-language basis.

## Substitution Targets

The following axioms have different implementations per language.

| axiom | Rust implementation | Substitution examples in other languages |
|---|---|---|
| `crypto.hash` | [neco-sha2](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [neco-sha1](https://github.com/barineco/neco-crates/tree/main/neco-sha1) | Python: `hashlib` (stdlib), Go: `crypto/sha256` (stdlib) |
| `crypto.secp` | [neco-secp](https://github.com/barineco/neco-crates/tree/main/neco-secp), [neco-p256](https://github.com/barineco/neco-crates/tree/main/neco-p256) | Python: `cryptography`, Go: `crypto/ecdsa` |
| `crypto.jwt` | Custom implementation | Python: `PyJWT`, Go: `golang-jwt/jwt` |
| `crypto.password` | Argon2id implementation | Python: `argon2-cffi`, Go: `golang.org/x/crypto/argon2` |
| `crypto.vault` | [neco-vault](https://github.com/barineco/neco-crates/tree/main/neco-vault) | Language-specific key management library |
| `json` / `cbor` / `kdl` | [neco-json](https://github.com/barineco/neco-crates/tree/main/neco-json), [neco-cbor](https://github.com/barineco/neco-crates/tree/main/neco-cbor), [neco-kdl](https://github.com/barineco/neco-crates/tree/main/neco-kdl) | Python: `json` (stdlib) / `cbor2`, Go: `encoding/json` |
| `str` (patch/fuzzy) | [neco-textpatch](https://github.com/barineco/neco-crates/tree/main/neco-textpatch), [neco-fuzzy](https://github.com/barineco/neco-crates/tree/main/neco-fuzzy) | Language-specific diff/fuzzy library |

## Substitution Steps

1. Add the mapping to the `bindings {}` section of `axiom/target/lang/{lang}/mapping.lex`.
2. Add a runtime call template to `rule.lex` as needed.
3. Run `laplan generate --target {lang}` and verify that import statements and package dependencies are correct.

To add a new language, see [adding-language.md](adding-language.md).

## The axiom Interface Is Common

Substitution only changes the implementation. The input/output types declared by an axiom (`data → digest`, `privkey, msg → signature`, etc.) are shared across all languages, and solver path resolution operates against this common interface. Differences in per-language implementations are resolved at synthesis output time and do not affect the solver.
