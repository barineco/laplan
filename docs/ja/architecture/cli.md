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
| `invert` | 既存コード → `.lex` 逆変換 (21 言語 + wasm) |
| `inverse` | (非推奨) Rust クレート → `.lex` 逆変換。後方互換のため残存、`invert` を使用してください |
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
| workspace モード | `--lib-lex` + `--face` 指定 | face の axiom member の lex/ を追加スキャン。face の capability を初期 marking に展開。`JointTransitionTable` を構築して cross-boundary transition も解析 |

### solve diagnose

全 endpoint の構造診断。`--face` は省略可能です。指定した場合は該当 face の cross-boundary endpoint も診断します。

```bash
laplan solve diagnose <dir> [--face <name>] [--lib-lex <path>] [--max-depth 8]
```

検出項目: MissingProduces, DeadBridge, SubtypeCycle, TimedCapabilityNoRenewal, LawTargetNotFound, ConvergentPaths。

ConvergentPaths は同一ゴールに異なる深さで到達する経路が複数存在する場合に報告する。

`--face` なし / `--lib-lex` なしでも動作します (dir モード)。`--lib-lex` を指定して `--face` を省略した場合は、全 face の統合診断を実行します。

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
| `--lib-lex <path>` | cratis/lib 宣言の取り込み |
| `--module-dir <dir>` | module.lex を含むディレクトリ |

詳細は [architecture/compiler.md](compiler.md) 。

## invert

既存コード (21 言語 + wasm) から `.lex` スケルトンを逆生成します。

```bash
laplan invert <target> <path> --namespace <ns> --output <file>
```

`<target>` には `axiom/target/lang/` 配下の言語名 (rust / typescript / python / ... の 21 言語) または `wasm` を指定します。`<path>` はディレクトリまたは単体ファイルです。

### 引数の判定フロー

| 条件 | 動作 |
|---|---|
| `<path>` の拡張子が `.wasm`、または `--format wasm` | `invert_wasm_binary` を呼ぶ |
| `<target>` が `rust` かつ `<path>` がディレクトリ | `invert_rust_crate` (内部で `invert_source_to_lex("rust", ...)` を呼ぶ thin wrapper) |
| `<target>` がその他の言語かつ `<path>` がディレクトリ | `cached_mapping(target)` で拡張子フィルタしたソースを収集し `invert_source_to_lex` |
| `<path>` が単体ファイル (wasm 以外) | ファイル 1 本を `invert_source_to_lex` に渡す |

### オプション

| オプション | 説明 |
|---|---|
| `--namespace <ns>` | 生成する `.lex` の namespace prefix (必須) |
| `--output <file>` | 出力先ファイルパス (必須) |
| `--format wasm` | 拡張子によらず WASM バイナリとして処理 |

### 動作例

```bash
# rust クレートの逆変換
laplan invert rust modules/pds/src --namespace pds --output out/pds.lex

# TypeScript ソースの逆変換
laplan invert typescript src/generated --namespace com.example --output out/types.lex

# WASM バイナリの逆変換
laplan invert wasm target/wasm/module.wasm --namespace com.example --output out/module.lex
```

逆変換の実装詳細は [architecture/compiler.md](compiler.md) の「laplan-inverse」節を参照してください。

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
