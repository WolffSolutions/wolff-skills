---
name: update-setup
description: Safely refresh a repo's agent setup — dependency bumps under a supply-chain release-age guard (bun/pnpm/npm/uv/cargo), plus vendored agent-skill updates via npx skills update with a lock-integrity audit that flags + offers cleanup of stale entries whose upstream source is deleted or renamed (404). Stack-aware. Use for an automated daily/weekly refresh, or whenever the user asks to "update deps", "bump packages", "update skills", "refresh setup", "clean up dead skills", "audit the lock", or when a build fails with "blocked by minimum-release-age". The maintenance counterpart to agent-setup.
metadata:
  author: MrBrunoWolff
  version: "2.2.0"
---

# update-setup — safely refresh deps + vendored skills

The maintenance counterpart to `agent-setup`: agent-setup *establishes* the setup (release-age
guards, vendored skills, hooks); this skill *refreshes* it without ever committing something the
deploy — or your supply-chain guard — will then reject.

Three jobs, each gated on what the repo actually uses:

1. **Dependencies** — bump under the package manager's minimum-release-age guard.
2. **Vendored skills** — `npx skills update` against `skills-lock.json`.
3. **Verify + commit** — frozen install, then stack-appropriate quality gates, commit only on green.

Run only the jobs that apply. A repo with a lockfile but no `package.json` still gets job 2.

## Why the release-age guard exists

A freshly published package version may be a supply-chain compromise that hasn't been caught yet.
A **minimum-release-age** guard refuses to install any version younger than N days (3 days /
259200s is the house default), so a malicious `x.y.z` published an hour ago can't land in your
lockfile — by the time it's installable, it's had days of public scrutiny.

The trap: a naive `--latest` rewrites a caret floor onto a brand-new version
(`"knip": "^6.15.0"` → `"^6.16.1"`), and if everything in range is inside the quarantine window
there is **no installable version left** — the deploy dies with
`blocked by minimum-release-age`. Every command below avoids that by keeping the guard active
*during* resolution, so `--latest` resolves to the newest version that is already past the window.

**Never lower or remove the guard to fix a failing build.** Fix the command, not the guard.

## Job 1 — Dependencies (per package manager)

Detect the manager from its lockfile and run the matching command. Multiple may apply (monorepo).

### bun — `bunfig.toml` / `bun.lock`

```bash
bun --version                                          # must be ≥ 1.3.x (older ignores the guard)
bun update --latest --minimum-release-age=259200
```

- `--latest` adopts newest versions across majors; `--minimum-release-age` keeps the 3-day guard
  explicit so it applies even if the runner's Bun doesn't read `bunfig.toml`.
- Guard lives **only** in `bunfig.toml` (`[install] minimumReleaseAge`, seconds). Bun does NOT read
  it from `.npmrc` — `min-release-age` there is an inert no-op.
- Bypass the wait for a trusted package via `minimumReleaseAgeExcludes = ["typescript", …]`.
- Known edge: `bun update --latest` can miss the guard for *transitive* deps
  ([oven-sh/bun#25305](https://github.com/oven-sh/bun/issues/25305)) — the frozen install in Job 3
  is the backstop.

### pnpm — `pnpm-lock.yaml`

```bash
pnpm update --latest
```

- Rolling guard: `minimumReleaseAge: 4320` (minutes = 3 days) in `pnpm-workspace.yaml`. Enforced on
  every install once set; if absent, `agent-setup` Phase 4 adds it.

### npm — `package-lock.json`

```bash
npx npm-check-updates -u && npm install     # or: npm update
```

- ⚠️ **npm has no rolling release-age guard.** The only native lever is `.npmrc` `before=<date>`,
  a *static* timestamp you'd have to bump by hand — not equivalent to bun/pnpm.
- Be explicit with the user: on npm, the supply-chain quarantine is **not** enforced. If they care,
  recommend migrating to bun or pnpm (both rolling). Do not pretend npm has parity.

### uv — `uv.lock` (Python)

```bash
uv lock --upgrade && uv sync
```

- Guard is `exclude-newer = "<RFC3339 date>"` in `[tool.uv]` — **static**. To honor a 3-day window,
  refresh it to `today − 3 days` before upgrading (this skill should bump that date, then upgrade).

### cargo — `Cargo.lock` (Rust)

```bash
cargo update
```

- No native release-age equivalent. Mention the gap; nothing to enforce.

## Job 2 — Vendored skills

If `skills-lock.json` exists, refresh all installed agent skills to their upstream sources:

```bash
npx skills@latest update -y
```

- This is the **only correct way** to change a vendored skill — hand-edits under `.claude/skills/`,
  `.agents/skills/`, etc. are overwritten here (the rule `agent-setup` enforces with a hook).
- Review the resulting `skills-lock.json` diff: changed `computedHash` values are expected; new or
  removed skills are worth a glance.
- ⚠️ `npx`/`bunx` run packages **outside** the lockfile release-age guard — `npx skills@latest`
  itself fetches the newest CLI. For fully reproducible automation, pin the version
  (`npx skills@<version>`).

### Handle stale lock entries (deleted + renamed upstream)

`npx skills update` does **not** prune a skill whose upstream source has disappeared or moved — it
prints `✗ Failed to update <name>` (when it tries) and leaves the entry stale in `skills-lock.json`
and on disk. Two signals feed cleanup, one reactive and one proactive:

**Signal A — reactive (CLI self-report).** Collect the `✗ Failed to update <name>` names from the
update output. Cheap, but **unreliable**: the same line fires on a transient network/rate-limit
error (you'll often see `✗ Failed to check for deleted skills from <source>` alongside it — the
CLI's own check being throttled), so a failure is *not* proof of deletion.

**Signal B — proactive lock audit (independent of the CLI).** A rename slips past Signal A: the old
path silently 404s without a clean "Failed to update" line, or the CLI's report is corrupted by
rate-limiting. So **independently verify every lock entry's path upstream**, not just the ones the
CLI flagged:

```bash
# for each entry in skills-lock.json: HEAD its source/skillPath
jq -r '.skills | to_entries[] | "\(.key)\t\(.value.source)\t\(.value.skillPath)"' skills-lock.json |
while IFS=$'\t' read -r name source path; do
  status=$(gh api "repos/$source/contents/$path" --jq '.path' 2>/dev/null && echo OK || echo GONE)
  [ "$status" = GONE ] && echo "STALE: $name ($source/$path)"
done
```

- Use `gh` (authenticated, ~5000 req/hr) so the sweep itself isn't what gets rate-limited. For a
  large lock, this is N requests — acceptable interactively; in fast/CI mode, gate it behind
  Signal A or an explicit "audit" request rather than running every time.
- `jq` may be absent — read the lock with whatever's present (`jq`/`node`/`bun`). Likewise the loop:
  prefer sequential checks; a broken `PATH` in a piped subshell can make every `gh` call silently
  fail.
- **Only a confirmed HTTP 404 means stale.** A `gh` call that errors for *any other reason* — 403,
  network, rate-limit, or the command failing to execute at all — is "couldn't verify," never
  stale. If the whole sweep comes back "all stale," that is a false-positive cascade (the checker
  is failing, not the skills) — abort the audit and report, don't prune anything.

**Confirm, then prune.** Union the two signals and classify each candidate:

1. Re-verify with a hard upstream check — a **404** = genuinely gone/renamed; `403`/network =
   transient, leave it and report "couldn't verify".
2. **Only for confirmed 404s**, ask the user (one prompt, multi-select) whether to clean each up.
   Cleanup = remove the entry from `skills-lock.json` **and** delete its installed dirs/symlinks
   across every agent store (`.claude/skills/<name>`, `.agents/skills/<name>`, `.cursor/…`).
   Pruning the lock matters: `npx skills remove` deletes files but does **not** prune the lock, so a
   leftover entry gets reinstalled (or re-fails) on the next update.
3. For a confirmed **rename** (the user has the new name installed, e.g. `update-deps` → the new
   skill), this is just a stale-entry cleanup — prune the old entry; the new one is already tracked.

Default to **keeping** anything uncertain. Deletion is the user's call, surfaced as a prompt — never
an automatic side effect of an update.

## Job 3 — Verify + commit

1. **Frozen install** — prove it installs the way the deploy will (no re-resolution):

   ```bash
   bun install --frozen-lockfile      # pnpm: pnpm install --frozen-lockfile · uv: uv sync --locked
   ```

   Must succeed / report "no changes". If it fails, **do not commit** — lock and manifest are out
   of sync.

2. **Stack-appropriate quality gates** — run what the repo defines, by stack:

   | Stack | Typical gates |
   |-------|---------------|
   | web / node | `lint` → `build` → `doctor`/tests (from `package.json` scripts) |
   | python | `ruff check` → `pytest` (or `uv run pytest`) |
   | rust | `cargo clippy` → `cargo test` |
   | go | `go vet ./...` → `go test ./...` |

   Skip gracefully if a gate isn't defined; never invent one.

3. **Commit only on green.** Commit the manifest + lockfile together; commit `skills-lock.json`
   separately (or in the same commit, clearly described) so a skill bump is distinguishable from a
   dependency bump in history.

## Scope

- Updating, not setting up — if a release-age guard is **missing**, that's `agent-setup` Phase 4,
  not this skill (though this skill should flag the absence).
- One repo at a time. For multi-repo refresh, the caller loops; this skill operates on the cwd.
