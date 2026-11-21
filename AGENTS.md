# AGENTS.md

Instructions for coding agents working in this repository.

## How this project is built

- Use EGM itself, standard flow: `egm-brainstorm` → `concepts_brief.md` → `egm-draft` →
  `design_draft.md` → `egm-review` → `egm-resolve`. Implement only after the owner approves the
  design.
- Run this project's own EGM process with native subagents. Do not use herdr, a prototype or
  this adapter to review this project's own documents.
- The current state and next steps are in issue #1 and in
  `docs/brainstorm/concepts_brief.md` (start with "Key Decisions & Staging for Next Steps").
- EGM artifact directory: `docs/designs` for the draft and review reports of the v1 design.

## Conventions

- Code and docs in English. Plain, direct language.
- Conventional Commits.
- Nothing machine-specific in the repository: no absolute home paths, personal deny rules,
  private harness details or credentials. Local settings live in the user's config directory.

## Owner gates (stop and ask)

- The first push of a new branch to `main`, and any force push.
- `herdr plugin install` or `link`, and anything that writes into another tool's config
  (`~/.claude`, `~/.gemini`, `~/.config/opencode`, ...), including installing EGM into `agy`.
- Planting test files in user-level config (for example a user `CLAUDE.md` canary).
