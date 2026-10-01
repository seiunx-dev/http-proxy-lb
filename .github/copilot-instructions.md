# Copilot Instructions

## Project coding notes

- Language: Rust (edition 2021)
- Runtime: Tokio
- Proxy protocol handling is in `src/proxy.rs`
- Configuration is YAML via `yaml_serde`
- Admin endpoints and shared metrics are in `src/admin.rs`
- Tests are unit tests next to the code (`#[cfg(test)]` modules in `src/`)
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
