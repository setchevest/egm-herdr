# Feature concept: egm-herdr (generic EGM × herdr adapter)

Owner: Sebas Etchevest (GitHub `setchevest`). Date: 2026-11-21.
Working directory (future repo root): the repository root (empty at the time).

## Anchored hypothesis

A new public repository (`setchevest/egm-herdr`, Apache-2.0, with attribution to
`mcorreiam/egm-herdr`, which is MIT) lets the EGM (Elephant-Goldfish Model) Elephant run in
Claude Code or in the Antigravity CLI (`agy`), and runs each Goldfish reviewer as a fresh,
isolated agent session in its own herdr pane. Each reviewer runs in kind `claude` or kind `agy`,
chosen by a reviewer → {kind, model} matrix and by per-kind launch profiles that live outside
the repo code (data, not logic). EGM personas, document standards and skill text are not forked:
they are read from the installed upstream EGM plugin. The adapter replaces only *where* a
Goldfish runs and *how* its report comes back.

## Why (the user's stated problems)

1. Run Goldfish reviewers on different harnesses and models. No harness can natively spawn a
   subagent of another vendor; herdr (a terminal multiplexer that runs and observes coding
   agents in panes and exposes a CLI to drive them) can.
2. Later (not v1): lower cost and use local or small/older models (glm, qwen, others) through a
   private company harness that already has a herdr integration (a herdr plugin with hooks that
   reports agent state). Later still: a "decision model" (JEV) may join the process.
3. Later (not v1): track the EGM process on GitHub (issues/PRs/comments).

## Success criterion for v1

A complete planning of a new idea (`egm-brainstorm` → `egm-draft` → `egm-review` →
`egm-resolve`) runs end to end twice: once with the Elephant in Claude Code and once with the
Elephant in agy. Both runs use a mix of `claude` and `agy` Goldfish reviewers. The human
intervenes only at EGM's human gates (no trust dialogs, no permission prompts, no stuck panes
to unstick by hand). "Isolated" is a test, not an impression: per kind, a smoke test proves
(a) no instruction files (CLAUDE.md, AGENTS.md, GEMINI.md, user-level memory) in the reviewer's
context, (b) reads allowed where intended and denied for secrets, (c) writes outside the report
location denied; plus a secret marker phrase that exists only in the Elephant's conversation and
that no reviewer can know. A kind is "supported" only after that smoke test passes on a real run.

## Decisions already taken by the owner

- New repo, not a fork of `mcorreiam/egm-herdr`. Public, under the personal GitHub account.
  License Apache-2.0; any copied MIT text keeps its notice in a NOTICE file.
- v1 kinds: `claude` and `agy`, for both roles (Elephant and Goldfish).
- Kind profiles and the matrix must be loadable from outside the repo (for example a user config
  directory), so a private company harness can be added later without touching the core and
  without putting private details in a public repo.
- Nothing machine-specific in the repo (the prototype's personal deny rules, home paths,
  homelab executor role move to local user config; the repo ships generic defaults/examples).
- Image/video generation and other non-review "worker" tasks are a separate future mechanism,
  not part of this adapter.
- Model routing/decision models (JEV), cost/local models, the company harness, GitHub tracking:
  out of v1. Requirement carried forward: reports and run state are files with a stable
  structure so a later GitHub layer can read them without touching the dispatcher, and the
  matrix is data an external tool could generate.
- How this project itself is built: standard EGM with native subagents. Do not use herdr or the
  prototype to run the EGM process of this project.

## Roles

- **Elephant**: the session that talks to the human, drafts briefs and design docs, dispatches
  reviewers, collects reports, stops at gates. Supporting a harness as Elephant needs: the EGM
  skills installed there, an adapter layer, a shell to call the dispatcher, and reliable
  obedience to the delegation override.
- **Goldfish**: a reviewer session launched in a pane, reads persona + target + standards,
  writes one report, ends. Supporting a harness as Goldfish only needs a kind profile (launch
  args, read-only mechanism, trust-dialog handling, report collection). A harness can be
  "Goldfish-only".

## Open design tension (not resolved)

Upstream EGM skills say "spawn a subagent with your harness's subagent tool". To dispatch to
herdr instead, something must change that instruction. `mcorreiam/egm-herdr` rewrote the
methodology in its own `SKILL.md` (a de facto fork of skill text that drifts with each EGM
release). The alternative under consideration: the Elephant follows the upstream skill
unchanged, and a thin adapter skill tells it only "where the skill says spawn subagent `egm-X`,
run `egm-herdr dispatch egm-X …` and read the report from the file". Unproven in any harness.

## Shape to challenge (from the owner's brief; not mandated)

Reviewer → {kind, model} matrix; one profile per kind (launch args, read-only mechanism, trust
handling, report collection: file or transcript); a dispatcher that only uses the herdr CLI; an
Elephant skill in the Agent Skills format; a `herdr-plugin.toml` for install and an optional
status pane.

## Facts validated on this machine (2026-11-21, herdr 0.9.5, Claude Code 2.1.311)

- A new pane's shell is not ready at once: `herdr agent start` returns `agent_pane_busy`; retry.
- Claude Code in a new folder shows a trust dialog whose default is "No, exit": send `down` then
  `enter`, only if "Yes, I trust this folder" is on screen.
- A prompt sent right after that dialog can sit unsent: `herdr agent prompt --wait --timeout
  15000` returns `agent_prompt_stalled`; `herdr pane send-keys <pane> enter` fixes it.
- Isolation that worked for kind `claude`: cwd in a fresh folder outside the user's projects
  tree (no CLAUDE.md/AGENTS.md discovered, empty auto-memory) + `--settings` with Read
  allow/deny rules + `--permission-mode dontAsk` + `--strict-mcp-config --no-chrome`. Verified:
  reads allowed where intended, `.env` denied, writes outside the allowed dir denied.
- `claude --bare` is not usable here: it requires an API key, and this machine uses OAuth.
- The Claude EGM reviewer adapters (`egm:egm-*`) are read-only (no Write). A Claude session's
  final message can be read exactly from `~/.claude/projects/<cwd with / and . as ->/<session-id>.jsonl`;
  the session id is `.agent_session.value` in `herdr agent get <name>`.
- herdr JSON: agent state is `agent_status` (`idle|working|blocked|done`).
- herdr plugin actions receive herdr context, not arguments: the Elephant still needs a skill and
  must call the dispatcher directly. A plugin is useful for install and an optional status pane.
- herdr integrations active here: `claude`, `copilot`, `antigravity-cli`. Not installed: `codex`.
- Upstream EGM reports instruction-file leaks into reviewer context for Codex and Antigravity
  (issue #237: project AGENTS.md and GEMINI.md); no host-specific exclusion control is verified
  for Antigravity.

## Findings from `mcorreiam/egm-herdr` (OpenCode + agy, docs in Portuguese)

- Its reviewers still load the project's AGENTS.md (documented limitation).
- `herdr agent wait` can return early for some kinds; it uses `herdr pane wait-output` on the
  reviewer's path reply as the primary trigger, then reads the report file (known a priori).
- `agent_prompt_stalled` can be a false negative (prompt delivered anyway).
- Reuses panes of completed stages for new reviewers; splits down when a pane would be under
  ~80 columns.
- agy reviewers use `--mode plan` for read-only (unverified) and model IDs that embed effort
  (`gemini-3.8-flash-medium`). First `agy` start in a workspace shows a trust dialog it says
  "only the user can answer".
- Pins upstream EGM by commit + SHA-256.
- About 15 of its 20 commits are OpenCode permission fixes; Elephant, commands and installer are
  tied to `~/.config/opencode`.

## Reference files (absolute paths; read as needed)

- Owner's brief: the owner's private brief (not in this repository)
- Working prototype (kind claude only): `.egm/refs/prototype-egm-herd.sh`
- `mcorreiam/egm-herdr` snapshot (commit in `COMMIT`): `.egm/refs/mcorreiam-egm-herdr/`
  (key files: `README.md`, `docs/design.md`, `skills/egm-herdr/SKILL.md`, `config/egm-herdr.json`, `agents/egm-reviewer.md`)
- herdr docs: `.egm/refs/herdr/`
  (`herdr-agent-skill.md` = CLI control; `herdr-plugins.mdx`, `herdr-marketplace.mdx` = plugin
  manifest/install/marketplace; `herdr-plugins.tsv` = 300 community plugins, none EGM-aware;
  closest: herdr-dagr, herdr-board, herdr-swarm)
- Upstream EGM installed plugin: `~/.claude/plugins/cache/egm/egm/0.1.1/`
  (`docs/harnesses.md`, `docs/claude.md`, `agents/`, `skills/`, `resources/document_standards/`)
