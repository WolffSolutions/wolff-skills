# Context-management playbook template

The block below is what `agent-setup` Phase 2 inserts into the repo's `CLAUDE.md` under
`## Context management`. Tailor the bracketed parts to the repo before inserting; drop rules that
reference skills the user declined in Phase 1.

---

## Context management

### Session hygiene

- **`/clear` between unrelated tasks.** A fresh session beats any amount of compaction. Never
  reuse a long conversation for a new feature or bug.
- **One task per session.** If the conversation drifts to a second concern, finish or hand off the
  first, then `/clear`.

### Compaction

- **Compact at natural boundaries, not under duress** — after a milestone lands, before starting
  the next chunk. Use a focus hint: `/compact keep the [API design decisions / migration plan]`.
- Auto-compact is the **safety net, not the strategy**. If the harness warns that context is
  getting low mid-task, prefer wrapping the current step and compacting deliberately over pushing
  until auto-compact fires mid-thought.

### Handoff

- **`/handoff` when ending a session mid-task, or when the chat has drifted** — produce the
  durable doc, `/clear`, resume from the doc. The handoff doc is the contract between sessions;
  the chat history is not.

### Delegation

- **Broad explorations go to subagents** (Explore/Task), not the main context. Searching many
  files for a conclusion should return the conclusion — file dumps must not enter the main
  conversation.

### Vendored skills are read-only

- **Never edit installed third-party skills** — anything under `.claude/skills/`, `.agents/skills/`,
  `.cursor/` (etc.) that is tracked in `skills-lock.json` is vendored from an upstream repo.
  Local edits are silently **overwritten by `npx skills update`** and were never yours to maintain.
- To change a skill's behavior: write your own skill in the project (different name), or PR the
  upstream source repo. Reviewers: treat diffs inside vendored skill folders as a red flag —
  either an agent misbehaved or an update ran; never hand-merge them.

### Durable knowledge

- **Decisions go to the repo, not the chat**: architecture decisions → [docs/adr/ or equivalent],
  conventions → this file. If something was hard-won in a conversation, write it down before
  `/clear`.
- **Keep this CLAUDE.md under ~150 lines** — commands and conventions only; link out for detail.
  Every line here is loaded into every session.

---

## Notes for the agent applying this (not part of the inserted block)

- If the repo already has a `## Context management` section, **replace** it (this template is the
  source of truth), but diff it for the user first — they may have local customizations worth
  keeping.
- Do not write percentage-based rules ("at 70% context do X") — there is no reliable context-%
  signal; the harness's own warnings are the trigger.
- If Phase 3b hooks were applied, mention `.claude/state/handoff.md` as the conventional handoff
  location so the SessionStart hook picks it up.
