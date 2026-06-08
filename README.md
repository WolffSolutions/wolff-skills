# wolff-skills

Personal [agent skills](https://skills.tools) by [@MrBrunoWolff](https://github.com/MrBrunoWolff),
installable with the [`skills`](https://github.com/vercel-labs/skills) CLI — the same tool used for
`vercel-labs/agent-skills`, `mattpocock/skills`, etc.

## Install

```bash
# a specific skill
npx skills@latest add WolffSolutions/wolff-skills --skill update-deps

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
| [`update&#8209;deps`](skills/update-deps/SKILL.md) | Safely update Bun dependencies (incl. new majors) without breaking the Vercel deploy under a `minimumReleaseAge` supply-chain guard. |

## Layout

```
skills/
  <skill-name>/
    SKILL.md      # YAML frontmatter (name + description) + the skill body
```

Each skill is a directory under `skills/` containing a `SKILL.md` with `name` and `description` in
its frontmatter — the format the `skills` CLI scans for.
