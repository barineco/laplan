# Generated Output License Specification

Files output by laplan carry either an MIT or MPL-2.0 license header depending on their origin. This specification defines the license boundaries for generated outputs. If an actual generated output differs from this description, report it as a bug.

## Licenses for Generated Outputs

| Output Type | License | Header |
|---|---|---|
| Regular modules (type declarations, handler skeletons) | MIT | `SYNTHESIZED_HEADER` |
| Runtime base (dispatch skeleton, boilerplate) | MIT | `SYNTHESIZED_HEADER` |
| WASM bindings (TypeScript / Python / server) | MIT | `SYNTHESIZED_HEADER` |
| Inverse output (code → `.lex` skeleton) | MIT | `INVERSE_HEADER` |
| Runtime solver (goal synthesis, path execution) | MPL-2.0 | Runtime solver-specific header |

### MIT and MPL-2.0 Boundary

- Outputs deterministically derived from `.lex` declarations and mapping.lex templates are **MIT**. They are derivatives of the user's data and contain no laplan-specific algorithms.
- Files whose generated content includes laplan's own solver logic (search algorithm, path verification, dispatch strategy) are **MPL-2.0**.

### Meaning for Users

- MIT outputs can be incorporated freely, with minimal license constraints on redistribution.
- If you modify an MPL-2.0 output (runtime solver), you are obligated to publish that file under MPL-2.0. However, MPL-2.0 does not propagate to surrounding MIT outputs or to the user's own code (file-level copyleft).

## Header Specification

All generated outputs carry a license header at the top of the file. The header has 4 lines containing:

1. SPDX license identifier (`SPDX-License-Identifier: MIT` or `MPL-2.0`)
2. Copyright line
3. Generation origin (generated via laplan)
4. Notes on modification and solver compatibility

Comment prefixes are automatically converted to match the target language:

| Language group | Prefix |
|---|---|
| Rust / C++ / Swift / TypeScript / Java, etc. | `//` |
| Haskell / OCaml | `--` |
| Python / Ruby / Elixir | `#` |
| Erlang | `%%` |

The meaning of the header body is preserved regardless of which prefix is used.

## Anomaly Criteria

The following states indicate a defect in the generation pipeline. Rewriting the header does not resolve them; the origin of the generated output must be reviewed.

| State | Expected normal behavior |
|---|---|
| A file with an MIT header contains solver logic | Solver logic is contained only in MPL-2.0 files |
| A file with an MPL-2.0 header contains only type declarations or decoders | Type declarations and decoders belong in MIT files |
| An inverse output carries an MPL-2.0 header | Inverse outputs always use MIT (`INVERSE_HEADER`) |
| A generated output has no license header | All generated outputs carry a header |
| The boundary between `runtime.{ext}` (MIT) and `runtime_resolve.{ext}` (MPL-2.0) is unclear | MIT/MPL-2.0 is distinguishable by filename |

If you discover any of these states, report them as an issue.

---

## Design Rationale

The following is the rationale behind the boundary design decisions. Users do not need to read this to make license determinations.

### Decision Flowchart

Which license a generated output belongs to is determined as follows:

1. Does the output include laplan's own solver / dispatch / inference logic? → **MPL-2.0**
2. Can the output be deterministically derived from `.lex` + mapping.lex? → **MIT**
3. Is it only type helpers, encoders / decoders, or constant declarations? → **MIT**
4. Is it only the runtime dispatch skeleton (boilerplate) with no search logic? → **MIT (runtime base)**
5. Does the runtime accept a goal, synthesize a path, or perform path verification? → **MPL-2.0 (runtime solver)**

### Header Constant Placement

Defined as public constants in `compiler/ir/src/lib.rs`. Each emit routine prepends them to the output file.

| Constant | Purpose | License |
|---|---|---|
| `SYNTHESIZED_HEADER` | General synthesis outputs | MIT |
| `INVERSE_HEADER` | Inverse outputs | MIT |

The header for the runtime solver is emitted by `write_runtime_resolve` via the `atproto-server` feature and declares MPL-2.0.

The per-language comment prefix conversion is handled by `rewrite_header_prefix(header, prefix)` in `compiler/ir/src/lib.rs`.
