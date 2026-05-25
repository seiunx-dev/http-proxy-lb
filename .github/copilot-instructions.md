# Copilot Instructions

## Project coding notes

- Language: Rust (edition 2021)
- Runtime: Tokio
- Proxy protocol handling is in `src/proxy.rs`
- Configuration is YAML via `yaml_serde`
- Admin endpoints and shared metrics are in `src/admin.rs`
- Integration coverage lives in `tests/integration_smoke.rs`
- Container smoke coverage lives in `scripts/docker-smoke.sh`

## Project status

- This project is in a completed-v1 / release-ready state.
- Default to preserving current runtime behavior unless a task explicitly asks for a change.
- Regressions in proxy forwarding, response codes, admin APIs, metrics, graceful shutdown, request timeouts, connection limiting, and Docker deployment should be treated as important.

## Development expectations

1. Make surgical changes that directly address the request.
2. Add/adjust tests for changed behavior.
3. Keep docs aligned with user-facing configuration changes.
4. Validate with:
   - `cargo test`
   - `cargo clippy -- -D warnings`
   - `cargo fmt -- --check`
   - `./scripts/docker-smoke.sh` when Docker-related or deployment behavior changes
   - `./target/debug/http-proxy-lb --config config.yaml --check` when config behavior changes

## Validation focus

- Prefer integration coverage for:
  - config validation
  - real HTTP status propagation
  - request timeout behavior
  - connection limiting behavior
  - admin metrics/status counters
- Keep README, `AGENTS.md`, and this file aligned when the project’s validation workflow changes.

## Domain policy expressions

`domain_policy.domains` currently supports:

- `domain:example.com` (exact match)
- `suffix:example.com` (domain suffix match)
- `*.example.com` (suffix shorthand)
- `.example.com` (suffix shorthand)
- `example.com` (backward-compatible exact-or-suffix behavior)

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
