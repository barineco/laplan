# rule.lex residual special-call reference

`rule.lex` retains only a call spec (`special-calls { callCheckNeeds }` + `transition` + `tuple-format` + `guard.error-type`); most vocabulary has moved to the `mapping.lex` `expr {}` dual schema. The language-specific calls that could not be absorbed remain here as residuals.

Each residual entry is annotated with a `// residual-reason: <short reason>` comment directly above it in `axiom/target/lang/<lang>/rule.lex`.

## Shared residuals (21 languages)

| call | reason |
|---|---|
| `special-calls.callCheckNeeds` | Per-language `await` / `try` / `allocator` prefix differences cannot be distinguished by the mapping.lex call-open dual |
| `guard.error-type` | Per-language error type-name meta. Not foldable into `expr.*` |
| `transition.assign-with-error-check` | Statement-level, already stored in `variable.assign_with_error_check`; rule.lex retains it as forward-reference meta |
| `transition.error-raise` | Statement-level, per-language `panic!` / `throw` / `error` differences |
| `tuple-format` (top-level string) | `Expr::Tuple` formatting meta |

## Language-specific residuals

| language | call | reason |
|---|---|---|
| rust | `special-calls.resolve` | cratis-specific library call; axiom-external runtime-imports |
| rust | `special-calls.loadRecipes` | cratis-specific library call; axiom-external runtime-imports |
| ocaml | `guard.guard-suffix` | Guard wrapping meta stemming from OCaml's `(* ... *)` comment syntax (not foldable into expr) |
| cpp / csharp / go / java / kotlin / swift / zig | `guard.capability-check`, `guard.ownership-check` | Guard invocations carrying `try` prefixes and naming differences; folding into mapping.expr is deferred |

## Re-absorption candidates

- `callCheckNeeds`: attach a call-open + modifier tag (await/try/sync) to `MethodCall` and migrate into mapping.lex `expr.library-call`
- `guard.error-type`: move the type-name map into a mapping.lex `type-map {}` (currently held as meta inside the `guard {}` section)
- `guard.capability-check` / `guard.ownership-check`: introduce prefix tags to allow pushing them into mapping.expr
- rust-specific `resolve` / `loadRecipes`: promoting cratis runtime-imports into the axiom core itself would allow absorption
