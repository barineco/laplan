# Template Syntax: mapping.lex Specification

`axiom/target/lang/<lang>/mapping.lex` defines both emit and invert using the same templates. In the forward direction (generating source code from IR), placeholders in the template are expanded with values. In the inverse direction (recovering IR from source code), the same template serves as a matching pattern to extract values from source.

This bidirectionality is achieved through two families of notation covered in this document.

| Notation | Role | Location |
|---|---|---|
| `{name}` placeholder | Value expansion and capture. Fills variables in forward; captures source fragments in inverse | Universal across all language mapping.lex files |
| `«directive»` directive | Semantic marker indicating block boundaries syntactically. Removed from forward output; interpreted only by inverse | Currently only Python's `«dedent»` |

The following sections detail each notation and how forward / inverse correspond.

## 1. `{name}` Placeholder

`{name}` in a template is a variable reference used in both forward and inverse.

### Basic form

`{name}` corresponds to a single identifier. The `name` part refers to an IR field name (`name`, `type`, `value`) or a structure name (`Name` for display names), depending on the template's definition context.

In forward, the IR value is expanded:

```kdl
binding { emit "let {name} = {value};" }
```

This `binding` template is the `let` binding emit definition used in Rust's mapping.lex (`axiom/target/lang/rust/mapping.lex:64`). `{name}` and `{value}` are expanded with the corresponding IR fields, producing output like `let x = 1;`.

In inverse, the same template functions as a matching pattern against source, capturing the source fragments at `{name}` / `{value}` positions. The matched identifiers and expressions are returned to the IR.

### Case sensitivity

`{name}` and `{Name}` are distinct. A convention in mapping.lex separates display names from type names.

| Notation | Typical meaning |
|---|---|
| `{name}` | Field names or variable names (lowercase-initial) |
| `{Name}` | Type names or structure names (uppercase-initial) |

Which to use is determined per template according to the language's naming conventions (PascalCase / snake_case).

### List / block capture

Some placeholders receive entire lists rather than single values. The following names have fixed roles per template; writing `{args}` / `{params}` / `{body}` causes the corresponding structure to be correctly expanded and captured.

| Notation | Corresponding structure |
|---|---|
| `{args}`, `{params}` | Argument or parameter lists (separator is language-dependent) |
| `{body}`, `{then_body}`, `{else_body}` | Statement blocks |
| `{arms}` | Match arm lists |
| `{fields}` | Struct field lists |

### Implementation

Placeholder expansion is handled by `compiler/synthesis/src/template_engine.rs`. Inverse matching is handled by `compiler/inverse/src/ast_inverse/engine.rs`. Both interpret the same template, but the scan direction and output roles are symmetric.

## 2. `«directive»` Directive

`«...»` is a semantic marker enclosed in Guillemets (double angle brackets). It is removed from forward output and used by inverse for purposes such as block boundary detection.

### Current directives

Only one directive is currently supported:

| Directive | Belongs to | Role |
|---|---|---|
| `«dedent»` | Python's product template `close` | Treats a position where indentation decreases as the block end |

In Python's mapping.lex, it is used as follows (`axiom/target/lang/python/mapping.lex:15-19`):

```kdl
product {
  emit {
    visibility ""
    keyword "class"
    open "(TypedDict):"
    close "«dedent»"
    field-format "    {name}: {type}"
    empty-body "    pass"
  }
}
```

Unlike brace languages, Python has no symbol to mark block endings, so the position where a non-whitespace line appears at column 0 must be treated as the block end. `close "«dedent»"` communicates this behavior to inverse.

### Removal in forward

Directives are removed from forward output. Specifically, `strip_invert_directives` in `compiler/synthesis/src/template_engine.rs` strips markers where identifier-form content (consisting only of alphanumeric characters, `_`, and `-`) is enclosed in `«»`, while preserving Guillemets containing other content (e.g., `«arbitrary string»` that may appear in source literals).

### Interpretation in inverse

On the inverse side, `find_matching_close_directive` in `compiler/inverse/src/ast_inverse/engine.rs` branches on the directive name. Currently only `dedent` is implemented; when matched, `find_dedent_boundary` searches for a non-whitespace line at column 0 to determine the block end.

### Adding future directives

To add a new directive, two locations need changes:

1. Place `«new-directive»` in the relevant template of the target language's mapping.lex
2. Add a new branch to the `find_matching_close_directive` dispatch table

On the forward side, `strip_invert_directives` generically removes identifier-form content, so no additional changes are needed as long as the directive name consists only of alphanumeric characters, `_`, and `-`.

## 3. Forward and Inverse Correspondence

The mechanism by which the same template functions in both directions is expressed via the `pattern { derive-from "emit" }` declaration.

```kdl
binding { emit "{name}" pattern { derive-from "emit" } }
```

This `binding` is a definition used in the pattern section of Rust's mapping.lex (`axiom/target/lang/rust/mapping.lex:93`). The `pattern` section defines inverse matching, and `derive-from "emit"` instructs that "the matching pattern is derived from the emit template." This appears uniformly across all 21 languages' mapping.lex files.

Three benefits:

1. The same syntax need not be written twice for forward and inverse
2. Changing emit automatically updates the pattern
3. Forward / inverse inconsistencies are eliminated at definition time

`«directive»` is removed from forward output but persists in the derived pattern side. Emit and pattern are a pair derived from the same template; directives remain only on the inverse side.

## 4. Concrete Examples Side by Side

The two notation families correspond to differences in language block structure.

### Rust (brace-based, no directive needed)

```kdl
binding { emit "let {name} = {value};" }
```

In brace languages, block boundaries are syntactically explicit via `{` `}` and `;`, so `{name}` / `{value}` placeholders alone close both forward and inverse.

### Python (indent-based, directive needed)

```kdl
product {
  emit {
    keyword "class"
    open "(TypedDict):"
    close "«dedent»"
    field-format "    {name}: {type}"
  }
}
```

In indent languages, block endings have no symbol marker, so `close "«dedent»"` instructs inverse on the termination detection method. `«dedent»` does not appear in forward output (removed by `strip_invert_directives`).

## 5. Related docs

- IR layers and inverse pipeline: [../architecture/ir.md](../architecture/ir.md)
- Lex layer classification: [layers.md](layers.md)
- Forward emit implementation: [../architecture/synthesis.md](../architecture/synthesis.md)
- Target language list: [target-languages.md](target-languages.md)
