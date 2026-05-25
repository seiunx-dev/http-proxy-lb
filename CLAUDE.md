# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`http-proxy-lb` is an HTTP relay proxy that load-balances traffic across a pool of upstream HTTP proxies. It supports CONNECT tunneling (HTTPS), plain HTTP forwarding, passive/active health checking, hot config reload, domain-based routing policies, Prometheus metrics, and graceful shutdown. Written in Rust (edition 2021) on the Tokio async runtime.

The project is in a completed-v1 / release-ready state. Preserve existing behavior unless a task explicitly requires a change. Regressions in proxy forwarding, admin APIs, metrics, timeouts, connection limiting, or Docker deployment are high priority.

## Build and validation commands

```bash
cargo build                          # debug build
cargo build --release                # release build
cargo test                           # unit + integration tests
cargo clippy -- -D warnings          # lint (treats warnings as errors)
cargo fmt -- --check                 # format check
./scripts/docker-smoke.sh            # container smoke test
./target/debug/http-proxy-lb --config config.yaml --check  # validate config file
```

Run a single test: `cargo test <test_name>` (e.g. `cargo test round_robin_cycles_through_online`).

Integration tests (`tests/integration_smoke.rs`) spawn the actual binary and use a `TEST_MUTEX` for serialization -- they require the binary to be built first (`cargo build`).

## Architecture

All source lives in `src/` as a single binary crate. Each client connection runs in its own Tokio task. Upstream connections are established fresh per request (no upstream connection pooling).

- **`main.rs`** -- CLI (clap), startup, TCP accept loop, signal-based graceful shutdown, hot-reload loop (polls config file mtime on an interval)
- **`config.rs`** -- YAML config model (via `yaml_serde`/`serde`), loading, validation. Defines `Config`, `BalanceMode`, `DomainPolicyConfig`, `LimitsConfig`, `UpstreamConfig`
- **`upstream.rs`** -- `UpstreamPool` (mutex-guarded vec of `Arc<UpstreamEntry>`) with selection strategies (weighted round-robin, best-score, priority). `UpstreamEntry` tracks per-upstream state: online/offline (atomic), latency EMA, active connections, consecutive failures. `reload()` preserves state for URL-matched entries
- **`proxy.rs`** -- Request handler: parses HTTP/1.x via `httparse`, dispatches CONNECT tunnels or plain HTTP forwards, handles retry logic (up to `min(pool_size, 3)` attempts), passive health detection (marks upstream offline on failure), domain policy routing, keep-alive loop, body forwarding (content-length and chunked)
- **`health.rs`** -- Background active health checker: probes offline upstreams via captive HTTP request (`generate_204` through the proxy), marks them online on HTTP 204 response
- **`admin.rs`** -- Admin HTTP server with hand-rolled request parsing: `/metrics` (Prometheus text format), `/status` (JSON), `/health`. Global `Metrics` struct with atomic counters shared across the application

Key data flow: `main` accept loop -> per-connection Tokio task -> `proxy::handle_client` (keep-alive loop) -> `dispatch` -> `handle_connect` or `handle_http` -> upstream selection via `UpstreamPool::select` -> retry loop with passive health marking.

## Development guidelines

- Make surgical, focused changes. Update tests and docs for user-visible behavior changes.
- Keep `cargo test`, `cargo clippy -- -D warnings`, and `cargo fmt -- --check` green.
- Run `./scripts/docker-smoke.sh` when touching Docker or deployment-related code.
- Integration tests cover: config validation failures, direct HTTP forwarding with real status propagation, request timeout (504), connection limiting (503), admin metrics/status counters.
- YAML config uses `yaml_serde` (not `serde_yaml`). The crate is imported as `yaml_serde` in Cargo.toml.
- No external HTTP framework -- admin server and proxy protocol handling are hand-rolled over raw TCP with `httparse` for parsing.

## Git commits

All commit subjects must follow:

```text
[Type] Short description starting with capital letter
```

Allowed types:

| Type      | Usage                                                 |
|-----------|-------------------------------------------------------|
| `[Feat]`  | New feature or capability                             |
| `[Fix]`   | Bug fix                                               |
| `[Chore]` | Maintenance, refactoring, dependency or build changes |
| `[Docs]`  | Documentation-only changes                            |

Rules:

- Description starts with a capital letter.
- Use imperative mood: `Add ...`, not `Added ...`.
- No trailing period.
- Keep the subject at or below roughly 70 characters.
- **Agent attribution uses the standard Git `Co-authored-by:` trailer in the commit body, not a free-form `Agent:` line.** This makes GitHub render the co-author avatar on the commit page. The trailer must be on its own line, separated from the subject by a blank line, in the form `Co-authored-by: <Display Name> <email>`. Suggested values per agent:
  - Claude (any 4.x): `Co-authored-by: Claude Opus 4.7 <noreply@anthropic.com>` (substitute the actual model, e.g. `Claude Sonnet 4.6`, `Claude Haiku 4.5`)
  - Codex: `Co-authored-by: Codex <noreply@openai.com>`
  - Copilot: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`

Examples from this repo's history:

```text
[Feat] More features
[Fix] Fix some bugs and add more tests
[Chore] Update Dockerfile
[Chore] Bump softprops/action-gh-release from 2 to 3
```

## GitHub Actions workflows

Use the standardized workflow layout in `.github/workflows`:

- `ci.yml` runs on `main` pushes, pull requests targeting `main`, and manual dispatch.
- Rust CI order: `cargo fmt --all -- --check`, `cargo check --locked --all-targets`, `cargo clippy --locked --all-targets -- -D warnings`, then `cargo test --locked`.
- `release.yml` is the standard release build entrypoint. It runs on `v*` tags and manual dispatch, builds release artifacts, uploads them with `actions/upload-artifact`, and publishes GitHub Release assets on tag pushes.
- `docker.yml` is the standard Docker entrypoint. It runs on `main` pushes, `v*` tags, PRs that touch Docker/build inputs, and manual dispatch. PRs build only; non-PR runs push GHCR images with lowercase image names and Docker metadata tags.

Workflow maintenance rules:

- Keep workflow filenames and top-level names aligned: `CI`, `Release`, `Docker`, and optional package-specific names.
- Use `actions/checkout@v6`, `actions/setup-go@v6`, `actions/upload-artifact@v7`, `actions/download-artifact@v8`, `softprops/action-gh-release@v3`, and current Docker actions (`setup-buildx@v4`, `login@v4`, `metadata@v6`, `build-push@v7`).
- Keep `permissions` minimal: `contents: read` for CI/Docker build-only work, `contents: write` for release publishing, and `packages: write` only when pushing container images.
- Use workflow `concurrency` keyed by workflow name and ref, with release jobs using `release-${{ github.ref_name }}` and `cancel-in-progress: false`.
- Do not reintroduce legacy workflow names such as `rust-ci.yml`, `build.yml`, `release-build.yml`, `docker-build.yml`, or `docker-release.yml` unless a package-specific workflow already exists and is intentionally preserved.
