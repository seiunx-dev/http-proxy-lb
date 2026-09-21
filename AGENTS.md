# AGENTS.md

## Purpose

This repository contains `http-proxy-lb`, a Rust (edition 2021, Tokio async runtime) HTTP relay proxy with upstream load balancing over a pool of upstream HTTP proxies: CONNECT tunneling (HTTPS) and plain HTTP forwarding, passive and active health checks, hot config reload, domain policy routing, admin endpoints, Prometheus metrics, graceful shutdown, request limits, and container deployment assets.

## Status

- The repository is in a near-release / completed-v1 state.
- Prefer preserving the current behavior and validation baseline unless the task explicitly requires a change.
- Treat regressions in proxy behavior, admin endpoints, metrics, timeout handling, connection limiting, or Docker deployment as high priority.

## Scope

- Keep code changes minimal and focused.
- Prefer targeted tests first, then full validation.
- Preserve existing behavior unless the task explicitly requires a change.

## Local validation

- Build: `cargo build` (debug) or `cargo build --release`
- Run tests: `cargo test`; a single test with `cargo test <test_name>` (e.g. `cargo test round_robin_cycles_through_online`)
- Run lint checks: `cargo clippy -- -D warnings`
- Format code: `cargo fmt -- --check`
- Run container smoke test: `./scripts/docker-smoke.sh`, especially when touching Docker or deployment code
- Validate config file: `./target/debug/http-proxy-lb --config config.yaml --check`

`tests/integration_smoke.rs` spawns the actual binary and serializes through a `TEST_MUTEX`, so the binary must be built (`cargo build`) before those tests will pass.

## Code structure

- `src/config.rs`: configuration model and YAML loading
- `src/admin.rs`: admin server, `/metrics`, `/status`, `/health`, and shared metrics
- `src/upstream.rs`: upstream entry state + pool selection/reload
- `src/health.rs`: active health checking logic
- `src/proxy.rs`: CONNECT and HTTP forwarding logic
- `src/main.rs`: startup, accept loop, background tasks
- `tests/integration_smoke.rs`: end-to-end integration tests for config validation, timeouts, metrics, and connection limits
- `scripts/docker-smoke.sh`: local Docker smoke test helper

All source is a single binary crate. Each client connection runs in its own Tokio task, and upstream connections are established fresh per request (no upstream connection pooling).

- `config.rs` defines `Config`, `BalanceMode`, `DomainPolicyConfig`, `LimitsConfig` and `UpstreamConfig`, and owns loading plus validation.
- `upstream.rs` holds `UpstreamPool` (a mutex-guarded vec of `Arc<UpstreamEntry>`) with weighted round-robin, best-score and priority selection. `UpstreamEntry` tracks online/offline (atomic), latency EMA, active connections and consecutive failures; `reload()` preserves state for URL-matched entries.
- `proxy.rs` parses HTTP/1.x with `httparse`, dispatches CONNECT tunnels or plain HTTP forwards, retries up to `min(pool_size, 3)` attempts, marks an upstream offline on failure (passive health detection), applies domain policy routing, and handles the keep-alive loop and body forwarding (content-length and chunked).
- `health.rs` probes offline upstreams in the background with a captive HTTP request (`generate_204` through the proxy) and marks them online on an HTTP 204 response.
- `admin.rs` hand-rolls request parsing and serves `/metrics` (Prometheus text format), `/status` (JSON) and `/health`, backed by a global `Metrics` struct of atomic counters shared across the application.
- `main.rs` owns the CLI (clap), startup, the TCP accept loop, signal-based graceful shutdown, and the hot-reload loop that polls the config file mtime on an interval.

Key data flow: `main` accept loop -> per-connection Tokio task -> `proxy::handle_client` (keep-alive loop) -> `dispatch` -> `handle_connect` or `handle_http` -> upstream selection via `UpstreamPool::select` -> retry loop with passive health marking.

## Expectations

- Keep changes minimal and targeted.
- Update tests and docs for any user-visible behavior change.
- YAML config uses `yaml_serde` (imported under that name in `Cargo.toml`), not `serde_yaml`.
- There is no external HTTP framework: the admin server and proxy protocol handling are hand-rolled over raw TCP with `httparse` for parsing. Keep it that way.
- Prefer keeping `cargo test`, `cargo clippy -- -D warnings`, `cargo fmt -- --check`, and `./scripts/docker-smoke.sh` green before considering work complete.

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
