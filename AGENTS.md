# AGENTS.md

## Purpose

This repository contains `http-proxy-lb`, a Rust HTTP relay proxy with upstream load balancing, health checks, hot reload, domain policy routing, admin endpoints, Prometheus metrics, graceful shutdown, request limits, and container deployment assets.

## Status

- The repository is in a near-release / completed-v1 state.
- Prefer preserving the current behavior and validation baseline unless the task explicitly requires a change.
- Treat regressions in proxy behavior, admin endpoints, metrics, timeout handling, connection limiting, or Docker deployment as high priority.

## Scope

- Keep code changes minimal and focused.
- Prefer targeted tests first, then full validation.
- Preserve existing behavior unless the task explicitly requires a change.

## Local validation

- Run tests: `cargo test`
- Run lint checks: `cargo clippy -- -D warnings`
- Format code: `cargo fmt -- --check`
- Run container smoke test: `./scripts/docker-smoke.sh`
- Validate config file: `./target/debug/http-proxy-lb --config config.yaml --check`

## Code structure

- `src/config.rs`: configuration model and YAML loading
- `src/admin.rs`: admin server, `/metrics`, `/status`, `/health`, and shared metrics
- `src/upstream.rs`: upstream entry state + pool selection/reload
- `src/health.rs`: active health checking logic
- `src/proxy.rs`: CONNECT and HTTP forwarding logic
- `src/main.rs`: startup, accept loop, background tasks
- `tests/integration_smoke.rs`: end-to-end integration tests for config validation, timeouts, metrics, and connection limits
- `scripts/docker-smoke.sh`: local Docker smoke test helper

## Expectations

- Keep changes minimal and targeted.
- Update tests and docs for any user-visible behavior change.
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
