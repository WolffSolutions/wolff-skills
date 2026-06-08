---
name: update-deps
description: Safely update Bun dependencies in a project without breaking the Vercel (or any Bun) deploy. Use for an automated daily dependency bump, or whenever the user asks to "update deps", "bump packages", "update bun dependencies", or when a build fails with "blocked by minimum-release-age".
metadata:
  author: MrBrunoWolff
  version: "1.0.0"
---

# update-deps — safe Bun dependency updates

Projects using this skill ship a **supply-chain quarantine guard** in `bunfig.toml`:

```toml
[install]
minimumReleaseAge = 259200   # 259200s = 3 days
```

Bun refuses to install any package version published less than 3 days ago. Vercel installs with Bun
(`vercel.json` → `bunVersion`, `bun.lock`), so **the guard is enforced on every deploy**.

The whole point of this skill: keep dependencies fresh — **including new majors** — without ever
committing a version that the build will then reject.

## The one rule

**Always run `bun update` with the release-age guard active.** The correct command is:

```bash
bun update --latest --minimum-release-age=259200
```

- `--latest` adopts the newest versions **across majors** (not just within existing caret ranges).
- `--minimum-release-age=259200` makes the 3-day guard explicit, so it applies even if the runner's
  Bun doesn't pick up `bunfig.toml`. With the guard active, `--latest` resolves to *the newest version
  that is at least 3 days old* — majors included, just delayed by 3 days. Nothing fresher than the
  window ever lands in `package.json` or `bun.lock`.

Use **Bun ≥ 1.3.x** — older Bun ignores `minimumReleaseAge` and will write bleeding-edge floors that
the deploy then blocks.

## Why the naive command breaks the deploy

`bun update --latest` *without* the guard rewrites a caret floor onto a brand-new version:

```
"knip": "^6.15.0"   →   "knip": "^6.16.1"     # 6.16.1 published <3 days ago
```

`^6.16.1` means `>=6.16.1 <7` — every in-range version is inside the 3-day quarantine, so there is no
installable version left. The deploy dies with:

```
error: No version matching "knip" found for specifier "^6.16.1" (blocked by minimum-release-age: 259200 seconds)
Error: Command "bun install" exited with 1
```

This collides almost daily for fast-moving packages (`knip`, `@types/*`). The explicit
`--minimum-release-age` flag prevents it by never raising a floor past the window.

## Procedure

1. Confirm Bun version: `bun --version` (must be ≥ 1.3.x).
2. Run the update:
   ```bash
   bun update --latest --minimum-release-age=259200
   ```
3. Verify the result installs cleanly the way the deploy will (frozen, no re-resolution):
   ```bash
   bun install --frozen-lockfile
   ```
   This must succeed / print "no changes". If it fails, **do not commit** — the lock and
   `package.json` are out of sync.
4. Run the project quality gates (whatever `package.json` defines), e.g.:
   ```bash
   bun run lint
   bun run build
   bun run doctor     # react-doctor, if present
   ```
5. Only if all of the above pass, commit `package.json` + `bun.lock` together.

## Notes & gotchas

- The release-age guard lives **only** in `bunfig.toml` (`minimumReleaseAge`, in seconds). Bun does
  **not** read it from `.npmrc` — keys like `min-release-age` there are inert no-ops. Don't rely on
  `.npmrc` for the day-guard.
- To let a trusted package update instantly (bypass the 3-day wait), add it to
  `minimumReleaseAgeExcludes` in `bunfig.toml`:
  ```toml
  [install]
  minimumReleaseAge = 259200
  minimumReleaseAgeExcludes = ["typescript", "@types/bun"]
  ```
- Known Bun edge case: `bun update --latest` can miss the guard for *transitive* deps
  ([oven-sh/bun#25305](https://github.com/oven-sh/bun/issues/25305)). Step 3's frozen install against
  the committed lock is the backstop — never skip it.
- Never lower or remove `minimumReleaseAge` to "fix" a failing build. That defeats the supply-chain
  protection. Fix the command, not the guard.
