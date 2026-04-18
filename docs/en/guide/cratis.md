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
        boundary { from "cratis-a" to "cratis-c" via "http" }
    }
}
```

| Field | Meaning |
|---|---|
| `emit` | Target cratis to generate from this face |
| `axiom` | Cratis treated as axiom (given fact) in this face |
| `capability` | Initial capability for this face. The workspace-mode solver expands it into the initial marking |
| `bind` | Binding language for this face (optional) |
| `boundary` | Cross-cratis boundary communication spec |

`client { ... }` and `server { ... }` are shorthand for `face "client" { ... }` and `face "server" { ... }`.

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
