# pkg-anthropic

Anthropic Claude SDK for [MVL](https://github.com/LAB271/mvl_language).

Pure MVL — no `bridge.rs`, no `extern` blocks. Typed Messages API client with full IFC security. Built on `pkg.tls.https` for HTTPS transport and `std.json` for serialization.

## Install

```bash
mvl add github.com/mvl-lang/pkg-anthropic v0.1.0
mvl install
```

## Usage

```mvl
use std.env.{get_secret}
use std.ifc.{Secret}
use pkg.anthropic.{Claude, ModelId, Message, Role, Response,
                    AnthropicError, anthropic_error_msg}

partial fn main() -> Unit ! Net + Console + Env {
    match get_secret("ANTHROPIC_API_KEY") {
        None => println("error: ANTHROPIC_API_KEY not set"),
        Some(api_key) => {
            let client: Claude = Claude::new(api_key);
            match client.messages(
                ModelId::Sonnet4_6,
                [Message { role: Role::User, content: "What is MVL?" }],
                256,
            ) {
                Err(e) => println("error: ".concat(anthropic_error_msg(e))),
                Ok(resp) => {
                    let content: String = relabel trust(resp.content, "DISPLAY");
                    println(content)
                },
            }
        },
    }
}
```

### Multi-turn Conversation

```mvl
let client: Claude = Claude::new(api_key);
client.messages(
    ModelId::Haiku4_5,
    [
        Message { role: Role::System, content: "You are a helpful assistant." },
        Message { role: Role::User, content: "What makes MVL different from Rust?" },
    ],
    512,
)
```

### Custom Endpoint

```mvl
use std.net.{api_endpoint}

let client: Claude = Claude::with_endpoint(
    api_key,
    api_endpoint("https://my-proxy.example.com"),
);
```

## API

### Types

| Type | Description |
|------|-------------|
| `Claude` | API client holding `Secret[String]` key and `ApiEndpoint[String]` base URL |
| `ModelId` | Enum: `Opus4_6`, `Sonnet4_6`, `Haiku4_5` |
| `Message` | `{ role: Role, content: String }` |
| `Role` | Enum: `User`, `Assistant`, `System` |
| `Response` | `{ id, content, model, usage, stop_reason }` — all strings are `Tainted[String]` |
| `Usage` | `{ input_tokens: Int, output_tokens: Int }` |
| `AnthropicError` | Enum: `HttpError`, `JsonParseError`, `ApiError(Int, String)`, `MissingField` |

### Functions

| Function | Signature | Effect |
|----------|-----------|--------|
| `Claude::new` | `(api_key: Secret[String]) -> Claude` | pure |
| `Claude::with_endpoint` | `(api_key: Secret[String], base_url: ApiEndpoint[String]) -> Claude` | pure |
| `Claude::messages` | `(self, model, messages, max_tokens) -> Result[Response, AnthropicError]` | `! Net` |
| `model_id_string` | `(m: val ModelId) -> String` | pure |
| `anthropic_error_msg` | `(e: val AnthropicError) -> String` | pure |

## Security

### IFC Model

The compiler enforces information flow control at every boundary:

| Data | Label | Why |
|------|-------|-----|
| API key | `Secret[String]` | Credential — compiler prevents logging or exposure |
| Base URL | `ApiEndpoint[String]` | Capability token — audited provenance |
| Response content | `Tainted[String]` | External data — untrusted until `relabel trust()` |
| Response metadata | `Tainted[String]` | External data — same treatment |
| Request content | bare `String` | Your data — you trust it |

### Trust Boundaries

The SDK has exactly **two declassification points**, each with an audit tag:

1. **`ANTHROPIC-AUTH`** — API key released at the HTTP boundary (`relabel release`)
2. **`ANTHROPIC-RESPONSE-PARSE`** — response body trusted for JSON parsing (`relabel trust`)

Error responses use a separate tag `ANTHROPIC-ERROR-DISPLAY` and are truncated to 512 bytes to prevent unbounded allocation from hostile responses.

### Effect Requirement

`Claude::messages` requires `! Net` — the compiler ensures only functions declaring network effects can call the API. Pure functions cannot accidentally make API calls.

## Architecture

```
pkg-anthropic/
├── mvl.toml                # package manifest (no [native] — pure MVL)
└── src/
    └── anthropic.mvl       # SDK: types, serialization, Messages API
tests/
    └── anthropic_test.mvl  # 30 unit tests (pure functions, no network)
```

## Effects

All constructors and helpers are **pure** (no effects). Only `Claude::messages` requires `! Net`.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
