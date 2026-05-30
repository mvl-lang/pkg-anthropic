# Contributing to pkg-anthropic

Thank you for your interest in contributing to the MVL Anthropic SDK.

## Getting Started

1. Fork this repository
2. Create a feature branch: `git checkout -b feat/my-feature`
3. Make your changes
4. Run the type checker: `mvl check src/anthropic.mvl`
5. Run tests: `mvl test tests/anthropic_test.mvl`
6. Commit with a conventional message: `git commit -m "feat: add ..."`
7. Push and open a pull request

## Development Setup

You need the [MVL compiler](https://github.com/LAB271/mvl_language) installed:

```bash
# Build from source
git clone https://github.com/LAB271/mvl_language.git
cd mvl_language
cargo build
```

## Code Style

- Follow the MVL syntax conventions (see the [MVL cheat sheet](https://github.com/LAB271/mvl_language/blob/main/CLAUDE.md))
- All public functions must have doc comments (`///`)
- All functions must declare their effects
- Use `total fn` where possible; `partial fn` only when unavoidable
- This package is pure MVL — no `bridge.rs` or `extern` blocks

## Testing

Tests live in `tests/anthropic_test.mvl`.

```bash
mvl check src/anthropic.mvl          # type-check
mvl test tests/anthropic_test.mvl    # run tests
```

## License

By contributing, you agree that your contributions will be licensed under the Apache License 2.0.
