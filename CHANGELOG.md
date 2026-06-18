# Changelog

All notable changes to pkg-anthropic will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.2.0] - 2026-06-18

### Changed
- Graduated all eligible `fn` declarations to `total fn` with explicit `decreases` clauses on while loops (`encode_messages`, `build_request_body`, `extract_content_text`, `parse_response`)
- `partial fn` retained only for `Claude::messages` (calls `std.json.encode`/`decode` and `https_post`, which are partial)
- Updated Makefile: `MVL` variable falls back to `mvl` on PATH; added `coverage`, `prove`, `version` targets; improved guard-mvl error message

### Added
- `.openspec/adr/0001-api-design.md` — documents IFC security model, constructor design, and error model
- `.openspec/adr/0002-explicit-total-fn-annotation.md` — totality policy: all total functions carry explicit `total fn`, no implicit `total*` permitted

## [0.1.0] - 2026-05-30

### Added
- Initial release — Anthropic Claude SDK for MVL
- `Claude` client with `Secret[String]` API key and `ApiEndpoint[String]` base URL
- `ModelId` enum: `Opus4_6`, `Sonnet4_6`, `Haiku4_5`
- `Claude::messages` — typed Messages API with `! Net` effect
- `Message`, `Role`, `Response`, `Usage`, `AnthropicError` types
- Full IFC security model: API key as `Secret`, responses as `Tainted`
- System message extraction (max 1, per API spec)
- Error body truncation (512 bytes) for hostile response protection
- JSON serialization/deserialization via `std.json`
- HTTPS transport via `pkg.tls.https`
- 30 unit tests (pure functions, no network calls)
- Pure MVL — no `bridge.rs`, no `extern` blocks
