# axiom 実装の言語別差し替え

axiom は型と操作のインターフェースを宣言しますが、実装は言語ごとに異なります。Rust では neco-crates 系の pure Rust 実装が使われますが、Python では `cryptography` パッケージ、Go では標準ライブラリの `crypto` パッケージなど、各言語のエコシステムに適した実装に差し替えられます。

この差し替えは `mapping.lex` の `bindings {}` セクションで宣言的に管理されます。

## 仕組み

各言語のターゲット定義 (`axiom/target/lang/{lang}/mapping.lex`) に `bindings {}` セクションがあり、axiom の操作名を言語固有のパッケージとインポートパスに対応付けます。

```kdl
// axiom/target/lang/python/mapping.lex
bindings {
    "signature.verify.ES256K" package="cryptography" version="41.0" \
        import="cryptography.hazmat.primitives.asymmetric.ec.ECDSA"
    "signature.verify.P-256"  package="cryptography" version="41.0" \
        import="cryptography.hazmat.primitives.asymmetric.ec.ECDSA"
}
```

synthesis はこの宣言を参照して、生成コードのインポート文とパッケージ依存を言語ごとに組み立てます。

## 差し替えの対象

以下の axiom は言語ごとに実装が異なります。

| axiom | Rust の実装 | 他言語での差し替え例 |
|---|---|---|
| `crypto.hash` | [neco-sha2](https://github.com/barineco/neco-crates/tree/main/neco-sha2), [neco-sha1](https://github.com/barineco/neco-crates/tree/main/neco-sha1) | Python: `hashlib` (標準), Go: `crypto/sha256` (標準) |
| `crypto.secp` | [neco-secp](https://github.com/barineco/neco-crates/tree/main/neco-secp), [neco-p256](https://github.com/barineco/neco-crates/tree/main/neco-p256) | Python: `cryptography`, Go: `crypto/ecdsa` |
| `crypto.jwt` | カスタム実装 | Python: `PyJWT`, Go: `golang-jwt/jwt` |
| `crypto.password` | Argon2id 実装 | Python: `argon2-cffi`, Go: `golang.org/x/crypto/argon2` |
| `crypto.vault` | [neco-vault](https://github.com/barineco/neco-crates/tree/main/neco-vault) | 言語ごとの鍵管理ライブラリ |
| `json` / `cbor` / `kdl` | [neco-json](https://github.com/barineco/neco-crates/tree/main/neco-json), [neco-cbor](https://github.com/barineco/neco-crates/tree/main/neco-cbor), [neco-kdl](https://github.com/barineco/neco-crates/tree/main/neco-kdl) | Python: `json` (標準) / `cbor2`, Go: `encoding/json` |
| `str` (patch/fuzzy) | [neco-textpatch](https://github.com/barineco/neco-crates/tree/main/neco-textpatch), [neco-fuzzy](https://github.com/barineco/neco-crates/tree/main/neco-fuzzy) | 言語ごとの diff/fuzzy ライブラリ |

## 差し替えの手順

1. `axiom/target/lang/{lang}/mapping.lex` の `bindings {}` セクションに対応を追加
2. 必要に応じて `rule.lex` にランタイム呼び出しのテンプレートを追加
3. `laplan generate --target {lang}` で生成し、インポート文とパッケージ依存が正しいことを確認

新しい言語を追加する場合は [adding-language.md](adding-language.md) を参照してください。

## axiom のインターフェースは共通

差し替えで変わるのは実装のみです。axiom が宣言する入出力の型 (`data → digest`, `privkey, msg → signature` 等) は全言語で共通であり、solver の経路探索はこの共通インターフェースに対して行われます。言語ごとの実装の違いは synthesis の出力時に解決され、solver には影響しません。
