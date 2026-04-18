# Adding a Language

Steps to add SDK generation for a new language. Place declarative templates in `axiom/target/lang/{lang}/` and `GenericBackend` picks them up automatically. No language-specific code in the Rust core is needed (except for special cases).

## Directory structure

```
axiom/target/lang/{lang}/
├── mapping.lex    # type, syntax, control, and handler templates (required)
├── morph.lex      # language-specific implementations of morphisms (axioms) (required)
├── type.lex       # additional type declarations (optional)
└── `cli { ... }` block in `mapping.lex`  # for CLI synthesis (optional)
```

## Writing mapping.lex

Rust example:

```kdl
extension "rs"

keywords strict "type" "self" "fn" "struct" "enum" "trait" "impl" "pub" "use" ...
keywords builtins "String" "Result" "Type" "Object"
file-collisions "import" "package" "module" "type" "mod" "nul" "con" "prn" "aux"
allow-keyword-fields #false
identifier-escape prefix="r#"

syntax {
    product {
        visibility "pub"
        attribute "#[derive(Debug, Clone, PartialEq)]"
        keyword "struct"
        open "{"
        close "}"
        field-format "    pub {name}: {type},"
    }
    sum { /* ... */ }
    alias { format "pub type {Name} = {type};" }
    import-format "use crate::{gen}::{path::}::{Name};"
}

control {
    if-open "if {cond} {"
    else-open "} else {"
    if-close "}"
    for-open "for {var} in &{collection} {"
    for-close "}"
    fn-open "pub fn {name}({params}) -> {return_type} {"
    fn-close "}"
    module-open ""
    module-close ""
}

variable {
    binding "let {name} = {value};"
    mutable-binding "let mut {name} = {value}.to_owned();"
    assign "{target} = {value};"
    return "return {value};"
}

handler {
    handler-open "pub trait {HandlerName}: Send + Sync {"
    handler-close "}"
    method "    fn handle({MethodParams}) -> {ReturnType};"
    // ...
}
```

### Required sections

| Section | Purpose |
|---|---|
| `extension` | Output file extension |
| `keywords strict` | Reserved words (identifier collision check) |
| `keywords builtins` | Built-in types |
| `syntax { product, sum, alias }` | Type declaration templates |
| `control` | Control flow syntax |
| `variable` | Binding, assignment, and return |
| `handler` | Endpoint handler trait |

### Optional sections

| Section | Purpose |
|---|---|
| `functional` | Lex₁ path (functional languages only). let-in, match, lambda, fold, etc. |
| `lowering` | Lowering templates for `fst`, `snd`, `from-maybe`, etc. |
| `bindings` | External library mappings |
| `stub-template` | Stub code |
| `effect-*` | Representation of effect types and values |

### Template variables

| Variable | Meaning |
|---|---|
| `{name}`, `{Name}`, `{NAME}` | lowercase / PascalCase / uppercase |
| `{type}`, `{return_type}`, `{params}` | Types and signatures |
| `{cond}`, `{var}`, `{collection}`, `{value}`, `{target}` | Control flow substitutions |
| `{gen}`, `{Gen}` | Basename of `generated-subdir` (lowercase / PascalCase) |
| `{path::}`, `{path.}`, `{path/}` | Module path separators (`::`, `.`, `/`) |

## morph.lex

Declares how axiom morphisms (e.g. `i32.add`) are implemented in the language.

```kdl
morph "i32.add" {
    inline "({a} + {b})"
}

morph "str.concat" {
    inline "format!(\"{}{}\", {a}, {b})"
}
```

Template variables embed argument names (`a`, `b`, ...) and the return value variable.

## type.lex

Language-specific additional type declarations. May be empty.

## Functional languages

For languages that use the Lex₁ path (Haskell, OCaml, Gleam, Elixir, etc.), add a `functional {}` section to `mapping.lex`.

```kdl
functional {
    let-in "let {bindings} in {body}"
    match "case {target} of { {branches} }"
    lambda "\\{params} -> {body}"
    // ...
}
```

`has_functional_templates()` automatically branches between the Lex₁ and Lex₂ paths based on the presence of `functional {}`.

## Adding tests

### runtime_emit regression test

```bash
cargo test -p laplan-synthesis --lib runtime_emit
```

Renders existing axioms with the new language's templates and verifies the output matches expectations.

### For the Lex₁ path

```bash
cargo test -p laplan-synthesis --lib template_engine_fn
cargo test -p laplan-synthesis --lib runtime_program_fn
cargo test -p laplan-synthesis --lib write_runtime_resolve_fn
```

### Build verification

```bash
cargo check -p laplan-synthesis
cargo check -p laplan-synthesis --no-default-features
```

## Checklist

1. Place `axiom/target/lang/{lang}/mapping.lex`
2. Declare axiom implementations in `axiom/target/lang/{lang}/morph.lex`
3. Verify `all_mapping_names()` returns the new language
4. Verify `generic_backend_for_target("{lang}")` returns `Some`
5. Add expected output to the regression tests (`runtime_emit`)
6. Update docs:
   - Add the Capability level to [reference/target-languages.md](../reference/target-languages.md)
   - Update `guide/{lang}.md` in downstream projects (neco-atproto, etc.) as needed

## Special cases

The following targets require additional code outside `GenericBackend`.

| Target | Location |
|---|---|
| WASM binary | `compiler/synthesis/src/wasm_emit.rs`, `wasm_lower.rs` |
| WGSL shader | `compiler/synthesis/src/wgsl_emit.rs` |
| Python binding | `compiler/synthesis/src/bind_python.rs` |
| TypeScript binding | `compiler/synthesis/src/bind_typescript.rs` |
| Server implementation (atproto-server feature) | `compiler/synthesis/src/bind_server.rs`, `server_output.rs` |

These are emit-layer concerns, not language templates. See [architecture/synthesis.md](../architecture/synthesis.md) for details.
