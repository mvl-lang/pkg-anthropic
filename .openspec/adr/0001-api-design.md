# ADR-0001: API Design — Typed Messages Client with IFC Security Model

**Status:** Accepted
**Date:** 2026-06-18
**Context:** pkg-anthropic wraps the Anthropic Messages API in MVL. The design question is how to model API credentials, network endpoints, and untrusted response data in a way that the MVL type system enforces correct handling at compile time.

## Decision

**The Claude client uses `Secret[String]` for the API key, `ApiEndpoint[String]` for the base URL, and `Tainted[String]` for all response fields originating from the remote API.**

### Type assignments

| Value | MVL type | Rationale |
|---|---|---|
| API key | `Secret[String]` | Credential — compiler prevents logging or accidental exposure |
| Base URL | `ApiEndpoint[String]` | Capability token — provenance tracked, audit on use |
| Response `id`, `content`, `model`, `stop_reason` | `Tainted[String]` | Untrusted remote data — must be explicitly trusted before use |
| Request content | `String` | Caller-owned data, caller trusts it |
| Token counts | `Int` | Numeric, not a string sink |

### IFC declassification points

There are exactly two declassification points in the codebase:

1. **`release(self.api_key, "ANTHROPIC-AUTH")`** in `Claude::messages` — the key exits the `Secret` wrapper at the HTTP boundary, tagged `ANTHROPIC-AUTH`. This is the only place the key can be observed.
2. **`trust(resp.body, "ANTHROPIC-RESPONSE-PARSE")`** — the raw HTTP response body is trusted for JSON parsing. The resulting parsed strings are immediately re-tainted with `ANTHROPIC-RESPONSE`.

Callers that need to use response content must explicitly `trust(field, "TAG")` — this creates an audit trail in code review.

### Constructor design

`Claude::new(api_key)` uses the default endpoint (`https://api.anthropic.com`). `Claude::with_endpoint(api_key, base_url)` accepts a custom `ApiEndpoint[String]`, useful for testing with mock servers without hardcoding URLs. Both constructors are `total fn` — they perform no I/O and always terminate.

### Error model

`AnthropicError` is a sum type with four variants:
- `HttpError(HttpsError)` — transport layer failure
- `JsonParseError(String)` — malformed JSON from API
- `ApiError(Int, String)` — non-2xx HTTP status with truncated body (max 512 bytes)
- `MissingField(String)` — expected JSON field absent or wrong type

The 512-byte truncation on `ApiError` error bodies prevents hostile large-response attacks from inflating memory.

## Consequences

- Callers cannot accidentally log the API key — the compiler enforces this.
- Response content cannot be used as a string without an explicit `trust` call, ensuring all usage is visible in code review.
- The `! Net` effect annotation on `Claude::messages` ensures callers declare network intent; pure test helpers are free of the annotation.
- All serialization helpers (`encode_message`, `build_request_body`, `get_string_field`, `get_int_field`, `extract_content_text`, `parse_response`) are pure `total fn` — they can be tested without a network and verified by the prover.

## Connected to

- Issue #1020: IFC model for pkg-anthropic
- ADR-0002: Explicit `total fn` annotation policy
- pkg.tls.https: `HttpsResponse`, `HttpsError`, `https_post`
