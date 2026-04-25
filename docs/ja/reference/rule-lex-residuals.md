# rule.lex 残存 special-call リファレンス

`rule.lex` は call spec (`special-calls { callCheckNeeds }` + `transition` + `tuple-format` + `guard.error-type`) のみを保持し、語彙の大半は `mapping.lex` の `expr {}` dual schema に移っている。ここに吸収できなかった言語固有 call を残存として記録する。

各残存項目は `axiom/target/lang/<lang>/rule.lex` 内の該当 entry の直前に `// residual-reason: <短い理由>` コメントを付与している。

## 共通残存 (21 言語)

| call | 理由 |
|---|---|
| `special-calls.callCheckNeeds` | 言語別に `await` / `try` / `allocator` 前置詞差があり、mapping.lex の call-open dual では区別不能 |
| `guard.error-type` | 言語別 error 型名の meta 情報。`expr.*` には畳めない |
| `transition.assign-with-error-check` | stmt 系、`variable.assign_with_error_check` に格納済 (rule.lex 側は forward 参照用メタ保持) |
| `transition.error-raise` | stmt 系、言語別 `panic!` / `throw` / `error` 差 |
| `tuple-format` (top-level string) | `Expr::Tuple` の format 指定 meta |

## 言語固有残存

| 言語 | call | 理由 |
|---|---|---|
| rust | `special-calls.resolve` | cratis-specific library call、axiom 外 runtime-imports |
| rust | `special-calls.loadRecipes` | cratis-specific library call、axiom 外 runtime-imports |
| ocaml | `guard.guard-suffix` | OCaml の `(* ... *)` コメント構文由来の guard 囲み meta (expr に畳めない) |
| cpp / csharp / go / java / kotlin / swift / zig | `guard.capability-check`, `guard.ownership-check` | `try` 前置詞や名前命名差分を含む guard 呼び出し、mapping.expr への畳み込みは今後の課題 |

## 再吸収候補

- `callCheckNeeds`: call-open + modifier tag (await/try/sync) を `MethodCall` に付随させて mapping.lex `expr.library-call` へ移行する案
- `guard.error-type`: 型名 map を mapping.lex `type-map {}` に移す案 (現状は `guard {}` セクション内に meta として保持)
- `guard.capability-check` / `guard.ownership-check`: 前置詞 tag の導入により mapping.expr に寄せる余地あり
- rust 固有 `resolve` / `loadRecipes`: cratis runtime-imports を axiom 本体へ昇格すれば吸収可能
