# CLI リファレンス

laplan はビルド後に `laplan` バイナリとして利用できます。bin は `compiler/cli` (`laplan-cli` crate) から配布されます。

```bash
cargo run -p laplan-cli -- <command> [options]
```

## サブコマンド一覧

| サブコマンド | 役割 |
|---|---|
| `lint` | Layer 0 静的検査 |
| `solve <diagnose\|paths\|reachable\|goal\|structure>` | Petri net 状態空間分析 |
| `generate` | 多言語 SDK emit (proved bundle / lexicon-dir を入力) |
| `emit-wasm` | WASM バイナリ emit (`--bake` 系オプション対応) |
| `inverse` | Rust クレート → `.lex` 逆変換 |
| `publish-packages` | 多言語パッケージの一括出力 |
| `audit-packages` | 公開済みパッケージの audit |
| `convert` | `.json` / `.kdl` / `.lex` の相互変換 |

## lint

Lexicon 定義の静的検査 (Layer 0)。TransitionTable を構築せず、定義の well-formedness のみを検査する。

```bash
laplan lint <dir> [--format text|json]
```

### 検出項目

| 種別 | 説明 |
|------|------|
| OrphanOutput | output フィールドが他の endpoint の input に接続されていない |
| UnsatisfiedInput | input フィールドに対応する output を持つ endpoint がない |
| TypeConnection | output → input のフィールド名一致による型接続 (情報提示) |

### 終了コード

- 0: 問題なし
- 1: lint 検出あり

## solve

Petri net の状態空間分析。TransitionTable を構築し、rule の requires/produces ネットワークを分析する。

```bash
laplan solve <subcommand> <dir> [options]
```

共通オプション:
- `--format text|json` (デフォルト: text)
- `--max-depth <n>` (デフォルト: 8)
- `--lib-lex <path>` workspace cratis (lib.lex) のパス。face/axiom 解決を有効化
- `--face <name>` 対象 face 名。`--lib-lex` と併用

### 2 モード

| モード | 条件 | 動作 |
|---|---|---|
| dir モード | `--lib-lex` 未指定 | 指定ディレクトリのみを再帰スキャン。実験用 |
| workspace モード | `--lib-lex` + `--face` 指定 | face の axiom member の lex/ を追加スキャン。face の capability を初期 marking に展開 |

### solve diagnose

全 endpoint の構造診断。

```bash
laplan solve diagnose <dir> [--max-depth 8]
```

検出項目: MissingProduces, DeadBridge, SubtypeCycle, TimedCapabilityNoRenewal, LawTargetNotFound, ConvergentPaths。

ConvergentPaths は同一ゴールに異なる深さで到達する経路が複数存在する場合に報告する。

### solve paths

特定 endpoint の全到達経路を深さ別に表示する。

```bash
laplan solve paths <dir> <endpoint-nsid> [--max-depth 8] [--marking <json>]
```

異なる深さの経路が共存する場合、短い方に `[!]` マーカーを付与する。

```
Goal: [output:access-jwt]
Marking: (empty)
────────────────────
Depth 2 (1 route):
  [!] resolve_handle → issue_session_pair

Depth 4 (1 route):
  validate_handle → check_uniqueness → generate_did → issue_session_pair
```

### solve reachable

指定した marking から到達可能な全 fact を深さ付きで列挙する。

```bash
laplan solve reachable <dir> --from '{"handle":"","password":""}' [--max-depth 4]
```

### solve goal

endpoint に縛られない自由なゴール指定で経路を探索する。

```bash
laplan solve goal <dir> "output:account,output:access-jwt" [--from '{"handle":""}'] [--max-depth 8]
```

ゴールの書式: `<kind>:<value>` をカンマ区切り。

| kind | 役割 | 例 |
|------|------|-----|
| `output` | rule の produces/requires で宣言されるデータ | `output:did`, `output:access-jwt` |
| `capability` | 発火に必要な権限トークン | `capability:signing-key` |
| `capability_expired` | 期限切れの capability | `capability_expired:token` |
| `input` | 外部から供給する入力。rule は produces しない | `input:handle` |
| `self_key` | 所有権の証明 | `self_key:self.repo` |
| `selected` | ユーザーが選択した値 | `selected:collection` |

### solve structure

Petri net の構造的な問題を検出する。

```bash
laplan solve structure <dir>
```

| 検出項目 | 意味 | 解釈の例 |
|----------|------|----------|
| Orphan place | いずれかの rule が `produces` に持つが、他の rule の `requires`/`consumes` に現れない fact | 往々にして output 側に現れる。エンドポイントの最終出力であれば正常。そうでなければ下流の rule が未定義か、型名の不一致で接続が切れている |
| Dead place | いずれかの rule が `requires` に持つが、他の rule の `produces` に現れない fact | 往々にして input 側に現れる。外部から供給される初期入力であれば正常。そうでなければ上流の rule が未定義 |

## generate

多言語 SDK の emit。入力として proved bundle (`--input`) か lexicon-dir (`--lexicon-dir`) のいずれかを指定する。

```bash
laplan generate --input <proved-bundle.json> --output <dir> [--target <lang>]
laplan generate --lexicon-dir <dir> --output <dir> [--target <lang>]
```

`--target` に `axiom/target/lang/` 配下のディレクトリ名 (rust / typescript / python / ... の 21 言語) を指定する。省略時は全言語を生成。

## emit-wasm

WASM バイナリの emit。`--primitive` 単体モード / `--bake` モジュールモードの 2 通り。

```bash
laplan emit-wasm --primitive <path> --output <file>
laplan emit-wasm --bake [--simd] [--parallel] [--constant-time] \
    [--lib-lex <path>] \
    [--bind <typescript|python> --bind-output <dir>] \
    --module-dir <dir> --output <file>
```

| フラグ | 効果 |
|---|---|
| `--simd` | SIMD 最適化 |
| `--parallel` | ParallelDag を組み込み並列化 |
| `--constant-time` | 定数時間実行 |
| `--bind <typescript\|python>` | WASM に対するバインディングを生成 |
| `--server-output <dir>` | サーバ実装 stub を生成 |
| `--lib-lex <path>` | cratis/lib 宣言の取り込み |
| `--module-dir <dir>` | module.lex を含むディレクトリ |

詳細は [architecture/compiler.md](compiler.md) 。

## inverse

Rust クレートから `.lex` スケルトンを逆生成する (現状 Rust 固定)。

```bash
laplan inverse --crate <src-dir> --namespace <prefix> --output <path>
```

型宣言の逆変換は mapping.lex 駆動の `TemplateInverter` で全言語に対応していますが、CLI 入口は Rust ソースのみを受け付けます。

## publish-packages

多言語パッケージの一括出力。

```bash
laplan publish-packages [--server-output <dir>] [--wasm-output <path>] \
    [--lang-output name:dir ...]
```

## audit-packages

公開済みパッケージの audit (`atproto-server` feature で有効)。

```bash
laplan audit-packages [--lang-output name:dir ...]
```

## convert

`.json` / `.kdl` / `.lex` の相互変換。

```bash
laplan convert --input <file> --format <json|kdl|lex> [--with-rule]
```

| 入力 | 対応する `--format` |
|---|---|
| `.json` (Lexicon JSON) | `kdl`, `lex` |
| `.kdl` | `json`, `lex` |
| `.lex` | `json`, `kdl` |

`--with-rule` は rule 宣言を含む場合に追加情報を保持する。
