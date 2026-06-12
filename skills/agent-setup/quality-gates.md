# Quality gates — per-stack guardrails reference

Canonical "run before you commit" checks per stack. Phase 2b detects which of these the repo
**actually has** (scripts in `package.json`, config files, dev-deps), writes the real commands into
the repo's `CLAUDE.md` under `## Quality gates`, and flags any **core** gate that's missing so the
user can add it.

Two tiers per stack:
- **core** — always suggest/expect; a repo without it has a real gap worth flagging.
- **optional** — suggest only if already present, or offer to add.

Order matters: cheap/fast checks first (lint, typecheck) so the agent fails early before the
expensive ones (build, full test suite).

## Web / Next.js (the Vercel default)

| Gate | Tier | Command (use the repo's actual script if defined) | Detect via |
|------|------|---------------------------------------------------|------------|
| lint | core | `bun run lint` (oxlint / eslint) | `lint` script, `oxlint`/`eslint` dep, `eslint.config.*` |
| typecheck | core | `bunx tsc --noEmit` (or `bun run typecheck`) | `typescript` dep, `tsconfig.json` |
| react-doctor | core | `bunx react-doctor -y .` (or `bun run doctor`) | `react` dep — always relevant for React/Next |
| knip (dead code) | optional | `bun run knip` | `knip` dep / `knip.*` config |
| build | optional | `bun run build` | `build` script — heavier; gate before push, not every edit |

> The fast core loop is **lint → typecheck → react-doctor**. `knip` and `build` are heavier —
> recommend them before pushing/PR, not on every change.

## Node backend (express / fastify / nest)

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| lint | core | `bun run lint` | `lint` script, eslint/oxlint |
| typecheck | core | `bunx tsc --noEmit` | `typescript` dep |
| test | core | `bun test` / `bun run test` | `test` script, vitest/jest dep |

## Python

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| lint | core | `ruff check` | `ruff` in deps / `ruff.toml` / `[tool.ruff]` |
| format check | core | `ruff format --check` | same |
| typecheck | core | `mypy .` or `ty check` | `mypy`/`ty` dep, `[tool.mypy]` |
| test | core | `pytest` (or `uv run pytest`) | `pytest` dep, `tests/` |

## Rust

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| format check | core | `cargo fmt --check` | `Cargo.toml` |
| lint | core | `cargo clippy -- -D warnings` | `Cargo.toml` |
| test | core | `cargo test` | `Cargo.toml` |

## Go

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| format check | core | `gofmt -l .` | `go.mod` |
| vet | core | `go vet ./...` | `go.mod` |
| lint | optional | `golangci-lint run` | `.golangci.*` |
| test | core | `go test ./...` | `go.mod`, `*_test.go` |

## iOS / Swift

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| lint | optional | `swiftlint` | `.swiftlint.yml` |
| build | core | `swift build` or `xcodebuild build` | `Package.swift` / `*.xcodeproj` |
| test | core | `swift test` or `xcodebuild test` | `Tests/` / scheme |

## Android / Kotlin

| Gate | Tier | Command | Detect via |
|------|------|---------|------------|
| lint | core | `./gradlew lint` (+ ktlint/detekt if present) | `build.gradle*`, `.editorconfig` |
| test | core | `./gradlew test` | `build.gradle*`, `src/test/` |

## What Phase 2b writes into CLAUDE.md

A `## Quality gates` section like (web example, using the repo's real scripts):

```markdown
## Quality gates

Run before every commit (fail early — stop at the first red):

1. `bun run lint`
2. `bunx tsc --noEmit`
3. `bun run doctor`   # react-doctor

Before pushing / opening a PR, also:

4. `bun run knip`     # dead-code check
5. `bun run build`

Never commit with a failing gate. If a gate is genuinely wrong (false positive), fix the rule or
the config — don't skip the gate or weaken it to pass.
```

Tailor commands to what the repo defines; only list gates the repo actually has, and separately
tell the user which **core** gates are missing (e.g. "no typecheck script — add `tsc --noEmit`?").
