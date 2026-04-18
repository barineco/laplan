# CLI Reference

laplan is available as the `laplan` binary after building. The binary is distributed from `compiler/cli` (the `laplan-cli` crate).

```bash
cargo run -p laplan-cli -- <command> [options]
```

## Subcommand List

| Subcommand | Role |
|---|---|
| `lint` | Layer 0 static checks |
| `solve <diagnose\|paths\|reachable\|goal\|structure>` | Petri net state space analysis |
| `generate` | Multi-language SDK emit (proved bundle / lexicon-dir as input) |
| `emit-wasm` | WASM binary emit (supports `--bake` options) |
| `inverse` | Rust crate → `.lex` inverse conversion |
| `publish-packages` | Bulk multi-language package output |
| `audit-packages` | Audit of published packages |
| `convert` | Bidirectional conversion between `.json` / `.kdl` / `.lex` |

## lint

Static checks for lexicon definitions (Layer 0). Checks only well-formedness of declarations without constructing a TransitionTable.

```bash
laplan lint <dir> [--format text|json]
```

### Detected Issues

| Kind | Description |
|------|------|
| OrphanOutput | An output field is not connected to the input of any other endpoint |
| UnsatisfiedInput | No endpoint holds an output matching the input field |
| TypeConnection | Type connection via matching field names between output and input (informational) |

### Exit Codes

- 0: no issues
- 1: lint findings present

## solve

Petri net state space analysis. Constructs a TransitionTable and analyzes the rule requires/produces network.

```bash
laplan solve <subcommand> <dir> [options]
```

Common options:
- `--format text|json` (default: text)
- `--max-depth <n>` (default: 8)
- `--lib-lex <path>` path to the workspace cratis (lib.lex). Enables face/axiom resolution.
- `--face <name>` target face name. Used together with `--lib-lex`.

### Two Modes

| Mode | Condition | Behavior |
|---|---|---|
| dir mode | `--lib-lex` not specified | Recursively scans the specified directory only. For experimentation. |
| workspace mode | `--lib-lex` + `--face` specified | Adds face axiom member lex/ to the scan. Expands face capabilities into the initial marking. |

### solve diagnose

Structural diagnosis for all endpoints.

```bash
laplan solve diagnose <dir> [--max-depth 8]
```

Detected: MissingProduces, DeadBridge, SubtypeCycle, TimedCapabilityNoRenewal, LawTargetNotFound, ConvergentPaths.

ConvergentPaths is reported when multiple paths reach the same goal at different depths.

### solve paths

Shows all reachable paths for a specific endpoint, grouped by depth.

```bash
laplan solve paths <dir> <endpoint-nsid> [--max-depth 8] [--marking <json>]
```

When paths at different depths coexist, the shorter path receives a `[!]` marker.

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

Enumerates all facts reachable from the specified marking, with depth.

```bash
laplan solve reachable <dir> --from '{"handle":"","password":""}' [--max-depth 4]
```

### solve goal

Searches for paths using a free goal specification not bound to an endpoint.

```bash
laplan solve goal <dir> "output:account,output:access-jwt" [--from '{"handle":""}'] [--max-depth 8]
```

Goal format: comma-separated `<kind>:<value>`.

| Kind | Role | Example |
|------|------|-----|
| `output` | Data that rules produce/require | `output:did`, `output:access-jwt` |
| `capability` | Permission token required to fire | `capability:signing-key` |
| `capability_expired` | Expired capability | `capability_expired:token` |
| `input` | External input supplied by the caller. Rules do not produce it. | `input:handle` |
| `self_key` | Proof of ownership | `self_key:self.repo` |
| `selected` | Value selected by the user | `selected:collection` |

### solve structure

Detects structural problems in the Petri net.

```bash
laplan solve structure <dir>
```

| Detected | Meaning | Interpretation |
|----------|------|----------|
| Orphan place | A fact that is produced but never required/consumed | Often appears on the output side. Normal if it is the final output of an endpoint; otherwise the downstream rule is undefined or a type name mismatch has broken the connection. |
| Dead place | A fact that is required but never produced | Often appears on the input side. Normal if it is an initial input supplied externally; otherwise the upstream rule is undefined. |

## generate

Multi-language SDK emit. Specify either a proved bundle (`--input`) or a lexicon-dir (`--lexicon-dir`) as input.

```bash
laplan generate --input <proved-bundle.json> --output <dir> [--target <lang>]
laplan generate --lexicon-dir <dir> --output <dir> [--target <lang>]
```

Pass a directory name from `axiom/target/lang/` (rust / typescript / python / ... for all 21 languages) to `--target`. When omitted, all languages are generated.

## emit-wasm

WASM binary emit. Two modes: `--primitive` standalone mode and `--bake` module mode.

```bash
laplan emit-wasm --primitive <path> --output <file>
laplan emit-wasm --bake [--simd] [--parallel] [--constant-time] \
    [--lib-lex <path>] \
    [--bind <typescript|python> --bind-output <dir>] \
    --module-dir <dir> --output <file>
```

| Flag | Effect |
|---|---|
| `--simd` | SIMD optimization |
| `--parallel` | Embed ParallelDag for parallelization |
| `--constant-time` | Constant-time execution |
| `--bind <typescript\|python>` | Generate bindings for WASM |
| `--server-output <dir>` | Generate server implementation stub |
| `--lib-lex <path>` | Load cratis/lib declarations |
| `--module-dir <dir>` | Directory containing module.lex |

See [architecture/compiler.md](compiler.md) for details.

## inverse

Inverse-generates a `.lex` skeleton from a Rust crate (currently Rust only).

```bash
laplan inverse --crate <src-dir> --namespace <prefix> --output <path>
```

Type declaration inverse conversion is handled by `TemplateInverter` driven by mapping.lex and covers all languages, but the CLI entry point accepts only Rust source.

## publish-packages

Bulk multi-language package output.

```bash
laplan publish-packages [--server-output <dir>] [--wasm-output <path>] \
    [--lang-output name:dir ...]
```

## audit-packages

Audit of published packages (enabled with the `atproto-server` feature).

```bash
laplan audit-packages [--lang-output name:dir ...]
```

## convert

Bidirectional conversion between `.json` / `.kdl` / `.lex`.

```bash
laplan convert --input <file> --format <json|kdl|lex> [--with-rule]
```

| Input | Supported `--format` |
|---|---|
| `.json` (Lexicon JSON) | `kdl`, `lex` |
| `.kdl` | `json`, `lex` |
| `.lex` | `json`, `kdl` |

`--with-rule` retains additional information when rule declarations are present.
