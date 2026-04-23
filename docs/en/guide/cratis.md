# Writing a cratis

A cratis is laplan's package and workspace declaration. It groups `.lex` files and manages boundaries with external implementations through `provides`, `requires`, and `faces`.

## Single cratis

```kdl
cratis "encrypted-dm" version=1 {
  provides {
    procedure "com.example.dm.send"
    procedure "com.example.dm.receive"
  }
  requires {
    axiom "str.concat"
    axiom "crypto.encrypt"
  }
}
```

| Block | Meaning |
|---|---|
| `provides { procedure/query/record/... }` | Declarations this cratis exposes externally |
| `provides { axiom ... }` | Axioms this cratis provides |
| `requires { axiom ... }` | Axioms this cratis depends on |

## Workspace cratis

A cratis with `members` is a workspace. It groups multiple child cratis and separates them by face (client, server, etc.).

```kdl
cratis "my-app" {
  members {
    "atproto-client" path="client/cratis.lex"
    "atproto-server" path="server/cratis.lex"
  }
  faces {
    face "client" {
      emit "atproto-client"
      axiom "atproto-server"
    }
    face "server" {
      emit "atproto-server"
      axiom "atproto-client"
    }
  }
}
```

## Three forms of CratisSource

Each entry in `members` takes one of the following forms.

| Form | Syntax | Use case |
|---|---|---|
| Path | `path="client/cratis.lex"` | Local relative path |
| Builtin | `from="axiom/crypto"` | Built-in cratis bundled with laplan |
| GitHub | `from="github:owner/repo/path" hash="..."` | Fetched from GitHub (requires filesystem feature) |

These three source forms are supported.

## face

```kdl
faces {
  face "client" {
    emit "cratis-a"
    emit "cratis-b"
    axiom "cratis-c"
    bind "typescript"
    boundary {
      "com.example.account.create" emit="server" message="CreateAccountRequest"
    }
  }
}
```

| Field | Meaning |
|---|---|
| `emit` | Target cratis to generate from this face |
| `axiom` | Cratis treated as axiom (given fact) in this face |
| `capability` | Initial capability for this face. The workspace-mode solver expands it into the initial marking |
| `trust` | Trust level for this face. Defaults to `full` |
| `bind` | Binding language for this face (optional) |
| `boundary` | Cross-cratis boundary communication spec |

Face names are open-ended in the new `face "name" { ... }` form. The current constraints are:

| Rule | Meaning |
|---|---|
| Non-empty | `face "" { ... }` is invalid |
| Allowed characters | ASCII letters, digits, `-`, and `_` only |
| No duplicates | The same face name may appear only once in a cratis |

Reserved names include `client`, `server`, `pds`, `appview`, and `bgs`. These names may gain special runtime meaning, but the parse layer accepts them the same way it accepts any other valid face name.

`client { ... }` and `server { ... }` are shorthand for `face "client" { ... }` and `face "server" { ... }`. This legacy shorthand is limited to client and server. Any arbitrary face name must use the explicit `faces { face "browser" { ... } }` form.

### trust

Adding `trust="lexicon-only"` turns a face into a stub face.

```kdl
face "bsky-official-pds" trust="lexicon-only" {
  axiom "com.atproto"
  capability "pds-endpoint"
}
```

- `full` (default): normal laplan-aware face. The solver follows rules and internal paths
- `lexicon-only`: the solver does not inspect internal paths and assumes the face unconditionally produces the endpoint outputs declared by the Lexicon
- `message` mismatches against a stub face are downgraded to warnings
- synthesis emits a response validator API for stub-facing endpoints. If the runtime body does not match the Lexicon output, it returns `MismatchError`

## boundary

The `boundary` block declares which endpoints cross a face boundary and which face they target.

```kdl
boundary {
  "com.example.account.create" emit="server" message="CreateAccountRequest"
  "com.example.account.create" emit="client" message="CreateAccountResponse"
}
```

| Attribute | Meaning |
|---|---|
| `emit` | Target face name. Binary targets such as `wasm` are also allowed |
| `message` | Type name carried across that boundary (optional) |

Rules without `message` keep the previous emit-only behavior. When `message` is present, workspace solve runs a static validation pass first. Unknown type names and duplicate messages for the same pattern still abort solve. Request/response mismatches remain errors for normal faces, but are downgraded to warnings when the peer is a stub face.

## Classification criteria

| Question | Layer |
|---|---|
| Is this a type existence declaration? | type |
| Is this an input/output structure definition? | Lex₀ lex (lexicon) |
| Does the solver explore this as a transition? | Lex₁ solve (rule) |
| Is this used as evidence rather than explored by the solver? | Lex₂ constrain |
| Is this package or external connection management? | Lex₃ package (cratis / import) |

See [reference/layers.md](../reference/layers.md) for details.

## cratis file layout

```
project/
├── cratis.lex               # workspace
├── client/
│   ├── cratis.lex           # standalone cratis for face "client"
│   ├── rule.lex
│   └── ...
└── server/
    ├── cratis.lex
    ├── rule.lex
    └── ...
```

The parser interprets `cratis`, `members`, `faces`, `client`, and `server` blocks.

## cratis.lex placement in the repo

Each category directly under `axiom/` (`i32`, `crypto`, `algebra`, `datum`, etc.) has a standalone `cratis.lex` that declares package boundaries via `provides { axiom ... }` and `requires { axiom ... }`. `axiom/target/` is a parent workspace that groups the `lang`, `binary`, and `bind` group cratis.

`provides { axiom ... }` is package metadata and is independent from endpoint loading. Member resolution in `cratis.lex` and peer `.lex` file loading within the package directory are performed, but automatic expansion of `derives`, `const`, and `rule` is not.

For the full picture of axioms, see the axiom table in [reference/layers.md](../reference/layers.md) and the [reference/axiom-*](../INDEX.md) pages.
