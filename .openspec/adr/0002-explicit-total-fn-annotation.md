# ADR-0002: Explicit `total fn` Annotation Policy

**Status:** Accepted
**Date:** 2026-06-18
**Context:** MVL infers totality for functions that have no unbounded loops and no calls to `partial fn`. These show up as `total*` (implicit) in `mvl assurance`. The question is: should pkg-anthropic rely on inference or annotate explicitly?

## Decision

**All functions that are total must carry the explicit `total fn` keyword.** No implicit totality (`total*`) is permitted in source files.

Rationale:
- `mvl assurance` reports `N total fn (N explicit, 0 implicit)` — this is the target state.
- Explicit annotation is a contract: the author claims termination and exhaustiveness, and the checker verifies it. Implicit totality is a silent default that can be broken by adding a call to a `partial fn` without realising the impact.
- `total fn` on a function that calls `partial fn` is a compile error — this is the intended safety net.
- `partial fn` is already explicit; totality should be equally explicit.

## Application

| Function kind | Keyword |
|---|---|
| Pure constructors, builders | `total fn` |
| Exhaustive enum match | `total fn` |
| For/while loops with `decreases` | `total fn` |
| Calls only `total fn` callees | `total fn` |
| Calls `partial fn` or has unbounded loops | `partial fn` |
| Actor methods | no keyword (actor scheduling is partial by nature) |

Effect annotations are orthogonal: `pub total fn foo(...) -> T ! SomeEffect` is valid.

## Totality classification for pkg-anthropic

### `total fn` in `src/anthropic.mvl`

| Function | Justification |
|---|---|
| `default_api_url` | Pure constructor, no loops |
| `api_version` | Constant return, no loops |
| `anthropic_error_msg` | Exhaustive match over closed enum |
| `Claude::new` | Pure struct constructor |
| `Claude::with_endpoint` | Pure struct constructor |
| `model_id_string` | Exhaustive match over closed enum |
| `role_string` | Exhaustive match over closed enum |
| `encode_message` | Pure struct/map construction |
| `encode_messages` | While loop with `decreases msgs.len() - i` |
| `build_request_body` | While loop with `decreases messages.len() - i`, calls only total fns |
| `get_string_field` | Exhaustive match, no loops |
| `get_int_field` | Exhaustive match, no loops |
| `extract_content_text` | While loop with `decreases items.len() - i` |
| `parse_response` | Exhaustive match, calls only total fns |

### `partial fn` in `src/anthropic.mvl`

| Function | Justification |
|---|---|
| `Claude::messages` | Calls `std.json.encode` (partial), `std.json.decode` (partial), `https_post` (partial); validation loop is bounded by `messages.len()` but the `! Net` effect already implies partial semantics |

## `decreases` clauses for while loops

Every while loop in the package iterates `i` from `0` to `list.len()` with unit increment. The termination variant is `list.len() - i`, which is:
- Non-negative at loop entry (guard `i < list.len()` implies `list.len() - i > 0`)
- Strictly decreasing each iteration (`i` increases by 1)

This pattern is idiomatic for bounded list traversals in MVL.

## Test file mirrors

Per the mirror pattern (see ADR-0001 of pkg-http), test files re-declare source helpers without `total`. This is intentional: test-file mirrors are not verified separately by the assurance tool. Mirror functions keep their bare `fn` or `partial fn` annotation even when the source carries `total fn`.

## Consequences

- `make assurance` will report `0 implicit total` — use this as the gate.
- Any new function added without a totality keyword will appear as `total*` and fail the policy check.
- Reviewers should reject PRs that introduce implicit totality.
- Adding a call to `std.json.encode`/`decode` or `https_post` inside a `total fn` will cause a compile error, making the impact of new `partial fn` dependencies visible immediately.

## Connected to

- MVL Req 3 (Totality): `mvl assurance` REQ3 verifies this
- MVL Req 8 (Termination): `total fn` fns must have provable termination
- ADR-0001: API design — constructor functions are total, I/O functions are partial
