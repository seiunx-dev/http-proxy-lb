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
- Run lint checks: `cargo clippy --all-targets -- -D warnings` (as CI)
- Check formatting: `cargo fmt -- --check` (run `cargo fmt` to fix)
- Run container smoke test: `./scripts/docker-smoke.sh`, especially when touching Docker or deployment code
- Validate config file: `./target/debug/http-proxy-lb --config config.yaml --check`

## Code structure

- `src/config.rs`: configuration model and YAML loading
- `src/admin.rs`: admin server, `/metrics`, `/status`, `/health`, and shared metrics
- `src/upstream.rs`: upstream entry state + pool selection/reload
- `src/health.rs`: active health checking logic
- `src/proxy.rs`: CONNECT and HTTP forwarding logic
- `src/main.rs`: startup, accept loop, background tasks
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
- Prefer keeping `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt -- --check`, and `./scripts/docker-smoke.sh` green before considering work complete.
- Keep `README.md` aligned with user-facing configuration and validation-workflow changes.

## Tests

- All tests are unit tests in `#[cfg(test)]` modules next to the code in `src/`; there is no `tests/` directory or integration suite.
- Covered: config parsing/defaults/validation (`config.rs`), upstream selection and reload state (`upstream.rs`), domain policy matching and absolute-URI rewriting (`proxy.rs`), the captive `generate_204` probe (`health.rs`, using a local fake proxy), and admin counters (`admin.rs`).
- Not covered by automated tests: request forwarding and status propagation, request timeout (`504`), connection limiting (`503`), and byte counters. `scripts/docker-smoke.sh` only builds the image, starts it with a temporary `0.0.0.0` config and curls `/health` and `/status`.

## Domain policy expressions

`domain_policy.domains` entries are matched by `domain_matches` in `src/proxy.rs`:

- `domain:example.com` — exact match
- `suffix:example.com` — the domain itself or any subdomain
- `*.example.com` / `.example.com` — suffix shorthand
- `example.com` — backward-compatible exact-or-suffix match

Empty expressions (`domain:`, `suffix:`, `*.`) match nothing.

## Deployment gotchas

- `config.example.yaml` binds `listen` to `127.0.0.1:8080` and leaves `admin_listen` commented out. In a container both must bind `0.0.0.0`, and `admin_listen` must be set, because the `Dockerfile` `HEALTHCHECK` and the `docker-compose.yml` healthcheck curl `http://localhost:9090/health`.
- The `monitoring` profile in `docker-compose.yml` bind-mounts `./prometheus.yml`, which is not in the repository; the user must create it.
- Hot reload polls the config file's mtime every `reload_interval_secs` (0 disables it); a reload that fails validation keeps the current config. Only the upstream list is reloaded (`UpstreamPool::reload`); `listen`, `admin_listen`, `mode`, `health_check`, `domain_policy`, `limits`, `access_log` and `reload_interval_secs` are read once at startup and need a restart.

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

CI reuses the shared templates in
[`seiunx-dev/ci-templates`](https://github.com/seiunx-dev/ci-templates) at `@v1`.
The files in `.github/workflows` are thin callers:

- `ci.yml` (`CI`) runs on `main` pushes, pull requests targeting `main`, and manual
  dispatch: `rust-ci` (`cargo fmt --check`, `cargo clippy --all-targets -D warnings`,
  `cargo test`, all `--locked`), `docker` and `actionlint`.
- `docker` (root `Dockerfile`, `linux/amd64` only, GitHub Actions cache) does not wait
  for the tests. PRs build only; on `main` it runs in parallel with `rust-ci` and pushes
  the immutable `ghcr.io/seiunx-dev/http-proxy-lb:sha-<full sha>` and `:sha-<7 chars>` as
  soon as the build finishes. The `Docker tags` job (`docker-retag.yml`, after `CI OK`)
  then moves `:main` to that digest without rebuilding, so `:main` only follows commits
  whose `CI OK` passed. An arm64 image would compile Rust under QEMU; add it only with a
  `$BUILDPLATFORM` cross-compile.
- The aggregate job **`CI OK`** is the only required status check.
- `release.yml` (`Release`): bump the version in `Cargo.toml` in a PR → merge and wait
  for `CI OK` on `main` → push the tag `v<version>`. `release-gate` refuses a tag that
  differs from `Cargo.toml` and waits for `CI OK` on the tagged commit; then the
  binaries are built (tags only; assets `http-proxy-lb-v<version>-<target-triple>.tar.gz`
  for `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu` and `aarch64-apple-darwin`,
  each with a top-level folder of the same name holding the binary, `README.md` and
  `LICENSE`, plus `http-proxy-lb-v<version>-x86_64-pc-windows-msvc.zip` with those files at
  the root), the `main` image `:sha-<sha>` is promoted (re-tagged, not rebuilt) to
  `:<version>`, `:<major>.<minor>` and `:latest`, and the GitHub Release is published
  with `SHA256SUMS-<tag>.txt`. Manual dispatch is a dry run: it builds the binaries and
  publishes nothing.

Workflow maintenance rules:

- Use the shared templates first. Add custom jobs or steps only when a template
  genuinely cannot meet the project's needs, keep them in the thin caller files, and
  add a comment explaining why.
- Template bugs and missing features are fixed upstream in `seiunx-dev/ci-templates`
  (new `v1.x.y` tag), not worked around here.
- Keep top-level `permissions: contents: read`; grant `packages: write` / `contents: write`
  only on the job that needs it.
- Third-party actions in caller-side custom steps are pinned to a full commit SHA with a
  `# vX.Y.Z` comment; Dependabot (`github-actions`) updates them and the template refs.
