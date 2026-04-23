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

## bindings / package / runtime-base の 3 層連携

`bindings {}` は axiom の実装接続だけでは完結しません。3 つの mapping.lex セクションが連携して `.lex` → 言語コードの接続経路を構成します。

| セクション | 責務 | 例 (Rust) |
|---|---|---|
| `bindings {}` | axiom の射 (`xrpc.call` / `crypto.hash` 等) を外部 crate / 関数に接続 | `"signature.verify.ES256K" crate="k256" version="0.13" fn="k256::ecdsa::VerifyingKey::verify"` |
| `package {}` | 生成パッケージの manifest (Cargo.toml / package.json / requirements.txt 等) テンプレート。必要 dependency の宣言 | `[dependencies] k256 = "0.13"` を `manifest-template` 内で宣言 |
| `runtime-base {}` | 生成コード側の runtime primitive (型定義 / decode helper / error wrapper 等) を埋め込む | `UnknownValue` / `Bytes` / `BlobRef` 型の Rust 実装を embed |

`bindings` で指名した関数がコンパイル時に解決されるよう、`package` の manifest で dependency を追加し、`runtime-base` で共通型を揃える、という 3 層連携です。

## laplan スコープ外の接続先宣言禁止

laplan 本体の `bindings {}` / `package {}` / `runtime-base {}` は **汎用な外部ライブラリ** (crypto / json / xrpc 等) のみを接続対象とします。特定のアプリケーション固有のサーバ実装 (PDS サーバ、AppView、BGS 等) を laplan 本体で hardcode することは禁止です。

理由:

- laplan は汎用コンパイラ基盤で、消費者プロジェクトから呼び出して使われる
- 特定アプリの接続先を laplan 本体に書くと、他の消費者プロジェクトが巻き込まれる
- アプリ固有の接続は消費者側の `cratis.lex` と mapping overlay で宣言するのが正攻法

許容される laplan 本体の接続先の例:

- `k256` / `p256` / `sha2` / `argon2` など **汎用 crypto 実装**
- `serde` / `serde_json` / `neco-json` / `neco-cbor` など **汎用シリアライゼーション**
- `reqwest` / `hyper` / `ureq` など **汎用 HTTP client** (xrpc axiom の Rust 側接続先)

禁止される接続先の例:

- `neco-server` / `neco-appview` など **特定サーバ実装**
- `bsky-pds` / `mastodon-server` など **特定プラットフォーム固有実装**
- 特定プロジェクトの名前空間を持つ crate (消費者プロジェクトが使う場合は、消費者側で mapping を overlay する)

消費者プロジェクトで特定の接続先を使いたい場合は、その消費者プロジェクトの `cratis.lex` + workspace 設定で laplan の mapping を extend する (例: neco-atproto が `atproto_xrpc` 固有の binding を上乗せする場合、neco-atproto 側の mapping で宣言)。
