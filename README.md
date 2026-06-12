# wolff-skills

Personal [agent skills](https://skills.tools) by [@MrBrunoWolff](https://github.com/MrBrunoWolff),
installable with the [`skills`](https://github.com/vercel-labs/skills) CLI — the same tool used for
`vercel-labs/agent-skills`, `mattpocock/skills`, etc.

## Install

```bash
# a specific skill
npx skills@latest add WolffSolutions/wolff-skills --skill agent-setup

# list what's available
npx skills@latest add WolffSolutions/wolff-skills --list

# everything
npx skills@latest add WolffSolutions/wolff-skills --all
```

Works with `bunx` too (`bunx skills@latest add ...`). Skills install into every detected agent
directory (`.claude/`, `.agents/`, `.cursor/`, `.github/`) and are tracked in `skills-lock.json`.

## Skills

| skill&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | what it does |
|---|---|
| [`agent&#8209;setup`](skills/agent-setup/SKILL.md) | Make a repo agent-ready in five opt-in phases: stack-aware curated skill installs, a context-management playbook in `CLAUDE.md`, a permissions allowlist, deterministic hooks (incl. vendored-skill write protection), and supply-chain release-age guards. |
| [`update&#8209;setup`](skills/update-setup/SKILL.md) | Refresh a repo's setup: stack-aware dependency bumps under a minimum-release-age supply-chain guard (bun/pnpm/npm/uv/cargo) **and** vendored agent-skill updates via `npx skills update`. The maintenance counterpart to `agent-setup`. |

## Catalog

[`skill-sources.md`](skill-sources.md) — the vetted catalog of agent-skill source repos (general,
web, iOS, Android, …) that `agent-setup` consumes (live-fetched at run time, so catalog updates
reach installed copies without reinstalling). Not a skill itself — propose additions via PR.

## Layout

```
skills/
  <skill-name>/
    SKILL.md      # YAML frontmatter (name + description) + the skill body
```

Each skill is a directory under `skills/` containing a `SKILL.md` with `name` and `description` in
its frontmatter — the format the `skills` CLI scans for.
