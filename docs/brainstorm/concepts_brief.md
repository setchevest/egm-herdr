# egm-herdr: generic EGM × herdr adapter

* **Author:** Sebas Etchevest
* **Date:** 2026-11-21

## Overview

> **Note on references.** Paths under `.egm/refs/` point to a local snapshot that is not in this repository: `mcorreiam/egm-herdr` at commit `aad963a`, herdr 0.9.5 docs, and the owner's private prototype script. Line references to those files are kept for traceability.


`egm-herdr` is a proposed public repository (`setchevest/egm-herdr`, Apache-2.0, with attribution to `mcorreiam/egm-herdr`, MIT). It lets the EGM Elephant run in Claude Code or in the Antigravity CLI (`agy`). Each Goldfish reviewer runs as a fresh agent session in its own herdr pane, of kind `claude` or `agy`. A reviewer → {kind, model} matrix and per-kind launch profiles choose the kind and model. Both are data kept outside the repo code. Upstream EGM personas, standards and skill text are read from the installed plugin and are not forked. The adapter changes only where a Goldfish runs and how its report comes back.

**Current shape (after the spikes and owner decisions of 2026-11-21; see Key Decisions 9-14):**
the owner starts an Elephant with `egm-herdr elephant claude|agy`, which injects one override
file and, in Claude, denies the native subagent tool. The Elephant follows the upstream EGM
skills unchanged, takes upstream's direct-dispatch branch, and calls `egm-herdr` for every
reviewer. The dispatcher starts each reviewer headless (`claude -p` or `agy -p`) in its own herdr
pane through a wrapper that reports `working`/`idle` with `herdr pane report-agent`, waits with
`herdr agent wait`, checks exit code and a non-empty response, and writes the report file
itself; reviewers write nothing. claude reviewers meet the isolation tests; agy reviewers run at
a documented "measured leak" level.

The FeatureConcept is `docs/brainstorm/feature_concept.md`. Seven reviewer lenses read it, together with the prototype, the `mcorreiam/egm-herdr` snapshot, the herdr docs and the installed upstream EGM plugin (`~/.claude/plugins/cache/egm/egm/0.1.1/`).

Summary of the result:

- **The value holds, but more narrowly than stated.** Reviewers from other vendors are real and new. herdr is not what makes them possible, because any Elephant with a shell can start `agy` or `claude` as a child process. What herdr adds is panes you can watch, a uniform agent-state surface, and a path to the company harness.
- **The thin delegation override is supported by upstream text.** Upstream already says "If your harness cannot nest subagents, use direct dispatch". It has not been tested in any harness.
- **The v1 success criterion depends on three unproven things.** These are agy reviewer isolation, starting agy with nobody at the keyboard, and Elephant obedience to the override. On this machine, a user-level `~/.gemini/GEMINI.md` exists and agy allows reads outside its workspace. Claude isolation test (a) passed only because no `~/.claude/CLAUDE.md` exists on this machine.
- **Most reviewers point to one main design fork.** They recommend running reviewers headless (`claude -p`, `agy --print`) inside herdr panes, with the dispatcher capturing the output. That would remove trust-dialog key automation, prompt-stall fixes, transcript scraping, write access for reviewers, and inferring completion from agent state. It might also remove the herdr state hooks that produce `agent_status`.
- **Every lens recommends spikes before `egm-draft`.**

## Spike Results (2026-11-21)

After the first Goldfish pass, spikes 1 (agy isolation), 2 (headless in herdr panes) and 7
(Claude user-level exclusion) ran on this machine. Evidence:
`docs/brainstorm/spikes/2026-11-21-isolation-headless.md`. The Analysis below predates them; where they
conflict, this section wins.

- **Headless works in herdr panes for both kinds, and TUI driving is not needed.** `claude -p`
  and `agy -p` started through `herdr pane run` showed no trust dialog, returned the report on
  stdout and exited 0.
- **`agent_status` works for headless reviewers when the wrapper reports it.** herdr's hooks
  report only session identity; lifecycle state comes from screen detection, which has nothing to
  read in headless mode (first run: claude `unknown`, agy a false `idle`). With the wrapper calling
  `herdr pane report-agent --state working` before the reviewer and `--state idle` after it,
  herdr showed `working` for the whole run and `herdr agent wait --until idle --until done`
  returned in the same second the process exited, for both kinds. The same mechanism fits any
  future kind, including the company harness. Exit code plus a non-empty response stay the
  success check, because agy soft-deny exits 0 with an empty response.
- **Claude reviewers can pass isolation tests (a)-(c).** Fresh cwd, `--setting-sources
  project,local`, `--settings` with read allow/deny rules, `--permission-mode dontAsk`, read-only
  tools, `--strict-mcp-config --no-chrome`. Reads of `.env`, outside files and
  `~/.claude/projects` were denied (Grep too); every write was denied on disk; OAuth works.
  Without `--setting-sources`, user settings reach the reviewer (it answered in the user's
  language). `--setting-sources` also controls `CLAUDE.md` loading (shown for the project level);
  the user-level `CLAUDE.md` case is inferred, not tested, because planting one is a gate.
- **agy reviewers pass (b) and (c) only with explicit `--settings` denies, and fail (a).** The
  user-level `~/.gemini/GEMINI.md` always loads (no setting or flag disables it). With default
  settings the reviewer followed its pointer into a personal file outside the workspace.
  `--add-dir` loads the added folder's `AGENTS.md`/`GEMINI.md`, and cwd inside the project loads
  them too (#237). `--sandbox` covers terminal commands only. Web search and `invoke_subagent`
  cannot be turned off cleanly; `--settings` rejects every tool-toggle key tried.
- **agy headless soft-deny empties the report.** Any action that would need approval is denied
  with exit 0, `status: SUCCESS` and an empty response. Deny rules return errors and the run
  continues, but Deny > Allow, so "deny all but the allowed set" cannot be written. An agy
  reviewer that reads one file outside its exact allow list returns nothing.
- **A dedicated agy HOME needs its own OAuth login** (an empty `HOME` asked for sign-in).

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-design` | completed |
| `egm-codebase-details` | completed |
| `egm-zoom-out` | completed |
| `egm-comprehension` | completed |
| `egm-implementation-readiness` | completed |
| `egm-structural` | completed |
| `egm-inventive-catalyst` | completed |

## Analysis

Severity: 🔴 Blocker, 🟡 Major, 🟢 Nit. Lens names are shortened: design, codebase, zoom-out, comprehension, readiness, structural, catalyst. Path references: `UP` = `~/.claude/plugins/cache/egm/egm/0.1.1`, `.egm/` = `.egm/`.

### 1. Does the stated value survive?

**🟡 Partly. The core value is reviewer diversity across vendors, not herdr itself and not isolation.** (all seven lenses)
- The concept says "No harness can natively spawn a subagent of another vendor; herdr can" (`.egm/feature_concept.md:19-21`). Reviewers called this overstated. Any Elephant with a shell can run `agy --print` or `claude -p`.
- herdr adds three things nothing else does:
  - live panes a human can inspect;
  - a uniform agent-state surface;
  - the future company harness, which already reports to herdr.
- Recommendation: state the value as "model and vendor diversity, made visible, with a path to the company harness". Treat isolation as a secondary benefit to be measured, not claimed (design, zoom-out, catalyst, structural).

**🟡 Routing a claude Goldfish through herdr under a Claude Elephant gives up isolation that already works.** (design, codebase, zoom-out, comprehension, catalyst, structural)
- Native adapters already set `omitClaudeMd: true` and read-only tools (`UP/docs/claude.md:136-141, 162-164`).
- The pane version adds a trust dialog, OAuth session load and fragile report collection. It also leaks user-level `CLAUDE.md` wherever one exists.
- What herdr does add is enforced filesystem read/write rules. Upstream calls native isolation "context and tool isolation, not filesystem isolation" (`UP/docs/claude.md:162-164`).
- Recommendation: add a matrix value `kind: native`, meaning "use the harness's own subagent tool", and send only foreign kinds through herdr. The owner must decide whether native claude counts toward "a mix of claude and agy" (`.egm/feature_concept.md:30-31`).

**🟡 Measure the diversity claim before building much.** (zoom-out, structural)
- Run one agy reviewer by hand on an existing draft and compare its findings with the native claude reviewer.
- Optionally record findings per kind and model in run state, so the claim can be checked later.

### 2. Delegation override (the open tension)

**🔴 The override carries the whole design and is untested in any harness.** (all seven lenses)
- Upstream text supports a thin layer. Every dispatching skill says: "Spawn each subagent with your harness's subagent tool … Pass the prompt verbatim and wait for the report. If your harness cannot nest subagents, use direct dispatch instead of the synthesizer." (`UP/skills/egm-brainstorm/SKILL.md:29-36`, `UP/skills/egm-review/SKILL.md:21-28`, `UP/skills/egm-review-loop/SKILL.md:36-44`)
- Direct dispatch is described at `UP/skills/egm-review-loop/SKILL.md:128-133`.
- The override must handle five things:
  - **Synthesizer spawns.** Brainstorm and review-loop spawn `egm-synthesizer`, which then spawns the reviewers (`UP/skills/egm-brainstorm/SKILL.md:162-174`, `UP/skills/egm-review-loop/SKILL.md:117-127`). Read literally, "dispatch `egm-X`" would send the synthesizer into a pane. There it has no shell and cannot call the dispatcher. The override must say "never dispatch the synthesizer; this harness cannot nest; use direct dispatch" (comprehension, readiness, structural, codebase).
  - **`docs/claude.md`.** Claude skills tell the Elephant to read it first (`UP/skills/egm-review/SKILL.md:15`). Its Reviewer isolation section says "Spawn a new native Agent of the plugin's registered reviewer type" (`UP/docs/claude.md:131-134`). The override must explicitly replace that section, and the agy equivalent (`invoke_subagent`, `UP/docs/harnesses.md:95`).
  - **The stop guard.** "If your harness has no subagent tool, stop" (`UP/skills/egm-review/SKILL.md:28`). If the native Agent tool is denied for enforcement, the override must name `egm-herdr dispatch` as the subagent tool, or this guard fires (codebase, design).
  - **How the override is loaded.** EGM skills set `disable-model-invocation: true` and the Skill tool refuses them (`UP/docs/claude.md:20-25`). So an adapter skill is not loaded next to `/egm:egm-review` automatically.
  - **A/B instances.** These are optional (`UP/agents/synthesizer.md:40`). Cut them from v1 or define them.
- Recommendation:
  - Write the override as a mapping table, not one sentence.
  - Frame it as one new row in upstream's Delegation table (`UP/docs/harnesses.md:88-101`): "herdr: subagent tool = `egm-herdr dispatch`; named personas = yes; nesting = no" (structural, catalyst).

**🟡 How to deliver and enforce the override (three options proposed).**
- (a) **Adapter skill as entry point.** The user runs `/egm-herdr <stage>`, which states the mapping, then reads the upstream SKILL.md by path and follows it in the same thread. Upstream already uses this pattern for nested skills (`UP/docs/claude.md:22-24`) (structural, codebase).
- (b) **Launcher.** `egm-herdr elephant claude` starts `claude --append-system-prompt-file …`, so the user does not have to remember to load anything (design, catalyst).
- (c) **Host-level enforcement.** A session-scoped `--settings` file denies the native `Agent` tool for `egm:egm-*` types. A slip into native dispatch then fails visibly instead of silently. For agy, whether `invoke_subagent` can be disabled is unverified (design, catalyst).
- Secondary trade-offs:
  - Deny-rule syntax and `--append-system-prompt` behaviour need checking on Claude Code 2.1.311.
  - The Elephant ends up with two launch styles.
  - With direct dispatch, the Elephant deduplicates the reports in its own context, which costs Elephant context. Upstream allows this (structural, catalyst).

**🟡 Make "obedience" a check, not an impression.** (comprehension, readiness, structural, codebase)
- A run fails if the Elephant transcript has any native `Agent` or `invoke_subagent` call.
- There must be one dispatch record per reviewer the skill names.
- Two runs (`.egm/feature_concept.md:29-31`) are thin evidence of "reliable". Readiness proposes 3 of 3 runs per harness for the override spike.

**🟡 Rewriting the skill text has already drifted, which argues for the thin layer.** (all lenses that read the snapshot)
- mcorreiam numbers its stages 1-5 with a different makeup: comprehension → critic → zoom-out → readiness → consolidated (`.egm/refs/mcorreiam-egm-herdr/config/egm-herdr.json:84-125`).
- Upstream runs stages 5-9 (`UP/skills/egm-review/SKILL.md:287-303`):
  - 5 = zoom-out + inventive-catalyst
  - 6 = comprehension
  - 7 = critic via review-loop
  - 8 = intermediate zoom-out
  - 9 = readiness + codebase-details + style
- Its brainstorm list adds style and drops zoom-out (`config/egm-herdr.json:75-83`).
- Its reviewer key `egm-design-critic` does not match upstream's `egm-design` (`UP/agents/design_critic.md:2`).

**🟢 `egm-draft` and `egm-resolve` never dispatch Goldfish.** (comprehension, readiness, structural, codebase)
- Only brainstorm, review and review-loop do (`UP/skills/egm-draft/SKILL.md:261`; lite and bootstrap also spawn but are out of scope).
- State this in the success criterion. Draft and resolve only exercise the Elephant side.

### 3. Isolation per kind

**🔴 agy Goldfish isolation may not be achievable, and the v1 criterion has no fallback.** (all seven lenses)
- Upstream issue #237: project `AGENTS.md`/`GEMINI.md` leak into agy reviewers, and no exclusion control is verified (`UP/docs/harnesses.md:151-159`). Upstream's only recorded agy run used a temporary HOME and did not test instruction-file exclusion (`UP/TESTING.md:145`, codebase).
- Read-only is unverified. mcorreiam calls agy read-only "prompt-only until validated" and `--mode plan` unconfirmed (`.egm/refs/mcorreiam-egm-herdr/docs/design.md:102`, `README.md:120`).
- What an agy reviewer would get on this machine (zoom-out, codebase):
  - user memory `~/.gemini/GEMINI.md`;
  - context files from installed agy plugins;
  - a pre-allowed MCP tool from the user's settings and `allowNonWorkspaceAccess: true` (`~/.gemini/antigravity-cli/settings.json:2,7`).
- A fresh cwd removes project instruction files but not the user-level `GEMINI.md`.
- A temporary HOME would remove the user file. It would probably also remove OAuth credentials and the herdr state hook at `~/.gemini/config/hooks/herdr-agent-state.sh` (codebase, structural).
- No source documents path-level read deny for agy, so test (b), "reads denied for secrets", may be impossible.
- Recommendation:
  - Spike agy isolation first.
  - Decide the fallback before the spike. Proposed options:
    - (a) two support levels, "isolated" and "runs with a documented, measured leak";
    - (b) agy stays "experimental", and the matrix refuses it unless the user sets `allow_unsupported: true`;
    - (c) claude-only Goldfish in v1.
- Possible reuse: `quaywin/agys`, a herdr plugin for "multi-profile isolation … for Antigravity CLI … sandboxing" (`.egm/refs/herdr/herdr-plugins.tsv:174`) (codebase).

**🔴 Claude isolation test (a) passed by accident.** (design, codebase, zoom-out, comprehension, readiness)
- `~/.claude/CLAUDE.md` and `~/.claude/rules/` do not exist on this machine.
- A top-level `claude` session loads the user-level `CLAUDE.md` whatever the cwd. It also loads user `settings.json`, including hooks: `~/.claude/settings.json:43` runs the herdr state hook. User allow rules merge with `--settings`, so on another machine they could widen reads or writes.
- Plugin and skill listings also load.
- The adapter frontmatter (`omitClaudeMd`, `tools: [Read, Grep, Glob]`) applies only to native Agent spawns, not to a top-level process in a pane (`UP/.claude-plugin/agents/zoom_out.md:5,8`).
- Recommendation:
  - Spike `claude --agent egm:egm-<x>` (does `omitClaudeMd` apply to a main-thread agent?) and `--setting-sources` without `user`.
  - Record the trade-off: stripping user settings may also strip the herdr hook that produces `agent_status`.
  - Do not claim generic isolation from this machine's result.

**🔴 Reviewers can reach the Elephant's conversation without being told it.** (design, zoom-out, readiness, structural, catalyst)
- herdr injects `HERDR_ENV`, the socket and pane IDs into every managed pane (`.egm/refs/herdr/herdr-agent-skill.md:71-75`; `herdr-plugins.mdx:268`). A reviewer with a shell can run `herdr pane read <elephant-pane>`.
- Transcripts on disk are readable:
  - The Claude Elephant's transcript is in `~/.claude/projects/*.jsonl`. The prototype denies that path only for claude reviewers (`.egm/refs/prototype-egm-herd.sh:61`).
  - An agy reviewer with `allowNonWorkspaceAccess` could read it.
  - agy's own history store path is unknown.
  - herdr's own persisted scrollback, if any, is a third path (unverified).
- The marker test as written ("does the reviewer know the phrase?") passes trivially when no reviewer tried. It also needs a second prompt to a finished reviewer, which breaks the fresh-session rule (`.egm/refs/mcorreiam-egm-herdr/skills/egm-herdr/SKILL.md:181`).
- Recommendation:
  - Require "no shell tool" for every v1 reviewer kind.
  - Generic defaults deny every harness's transcript store for every kind.
  - If agy's shell cannot be removed, strip `HERDR_*` from the reviewer's environment, after checking this does not break the state hooks.

**🟡 Redesign the smoke test with canaries, controls and active probes.** (readiness, catalyst, design, codebase, comprehension)
- Use random per-run tokens (for example `EGMCANARY-<128-bit>`), one per channel:
  1. the Elephant conversation;
  2. project `CLAUDE.md`/`AGENTS.md`/`GEMINI.md` in a throwaway project;
  3. user-level instruction files (a throwaway HOME if OAuth allows; otherwise a gated temporary edit);
  4. the slot memory directory.
- Add a positive control: a token the reviewer must read and report, which proves the probe works.
- Add active probes. The reviewer must try each of these, and each must be denied:
  - read a `.env` canary, including through Grep and Glob, which Claude enforces only best-effort (`UP/docs/claude.md:101-102`);
  - read both harnesses' transcript folders;
  - run `herdr pane read` on the Elephant pane;
  - write anywhere.
- Check every outcome on disk or in the transcript. Never accept the model's own account.
- Pass threshold: 3 of 3 runs per kind.
- Save a dated evidence file per kind (CLI versions, profile hash, each probe's expected and actual result). "Supported" cites that file and is computed from it, not set by hand.
- Theoretical limit (catalyst): asking a model what it saw cannot prove a leak is absent. The smoke test is evidence, not proof. Say so in the docs.

**🟡 Use an allowlist for reads, not a blocklist.** (design, comprehension)
- The prototype allows all of `~/development/**` and denies a few patterns (`.egm/refs/prototype-egm-herd.sh:52-62`). `*.pem`, kubeconfigs, `tfstate` and `.npmrc` stay readable.
- Generate the allowed set per dispatch: persona, standards, target, GoldfishContext, plus the project root only when the persona needs code (codebase-details).
- Alternative, the **review bundle** (design, catalyst): copy the target, reading list, persona and standards into the reviewer's folder (APFS `cp -c` costs almost nothing). All reads stay inside the workspace and secrets cannot be reached.
  - Trade-off: path remapping in findings.
  - Trade-off: large bundles for codebase-details.

**🟡 OS-level sandbox as a harness-independent enforcement layer (catalyst suggestion).**
- macOS `sandbox-exec`, bubblewrap on Linux later. Isolation would then stop depending on each harness's own permission rules, which also covers the company harness.
- Trade-offs: `sandbox-exec` is deprecated and macOS-only, and may break OAuth keychain or network access.

### 4. "Human only at gates"

**🔴 A fresh folder per reviewer guarantees a trust dialog per reviewer, and agy's dialog is said to need the human.** (design, readiness, comprehension, catalyst, structural, codebase)
- Claude's trust dialog defaults to "No, exit" (`.egm/feature_concept.md:88-89`).
- mcorreiam: agy's first-start dialog is one "only the user can answer" (`.egm/refs/mcorreiam-egm-herdr/skills/egm-herdr/SKILL.md:60`).
- herdr's agent guidance says to ask the user before answering a blocked UI (`.egm/refs/herdr/herdr-agent-skill.md:145`). Automating the dialog is a deliberate exception and must be written down as one.
- Proposed options:
  - (i) **Stable slot folders, throwaway contents.** A fixed pool of folders per kind (for example `$XDG_CACHE_HOME/egm-herdr/<kind>/slot-N`), trusted once at a declared setup gate. The dispatcher wipes contents and that slot's memory directory before each run. Fresh context comes from a fresh process, not a fresh path (catalyst, structural, readiness). Trade-offs: memory may carry over if a wipe fails; wiping `~/.claude/projects/<slot>/memory` touches another tool's state, which is a gate; pool size caps parallelism.
  - (ii) **Key automation**, as in the prototype. Only for folders the dispatcher created, only on an exact screen match, and every automated keypress is logged.
  - (iii) **Headless mode**, which may skip the dialog entirely (see section 5).
- Caveat (zoom-out): agy's `trustedWorkspaces` includes the home folder. If trust matches by prefix, "no dialog" on this machine proves nothing about a clean machine.

**🟡 A `blocked` reviewer must never wait for the human.** (readiness, comprehension, structural, catalyst)
- Policy: answer only the known dialog. Treat any other `blocked` state as failure:
  - save the screen to run state;
  - kill the pane;
  - mark the reviewer `not run`;
  - let the Elephant apply upstream's retry-once rule (`UP/skills/egm-review-loop/SKILL.md:140-144`).
- mcorreiam asks the user before closing panes (`SKILL.md:91,179`), which is also human input outside the gates. Close panes automatically.

**🟡 The Elephant's own permission prompts and shell timeouts are missing from the concept.** (all lenses except structural raised at least one part)
- Every `egm-herdr` call is an Elephant shell call. Claude needs an allow rule such as `Bash(egm-herdr:*)`. agy's shell allowlist (`command(...)` in its settings) needs an entry too. Writing either rule into `~/.claude` or `~/.gemini` is an owner gate (owner's private brief).
- Reading personas from the plugin cache may need explicit read rules in headless runs (`UP/docs/claude.md:57-66`).
- Timeouts:
  - The prototype blocks up to 30 minutes per stage (`.egm/refs/prototype-egm-herd.sh:36`), and its `cmd_wait` waits N × timeout in the worst case (`:157-166`).
  - Reviewers report that Claude Code's Bash tool defaults to 2 minutes and caps at about 10. agy's limit is unknown. (Not verified on this machine.)
- Recommendation:
  - List the Elephant-side prerequisites per kind as one-time setup, outside "the run".
  - Reword the criterion to "no human action during a run; a documented one-time setup is allowed".

### 5. Execution mode: headless vs interactive TUI (main design fork)

**🟡 Run reviewers headless inside herdr panes; keep TUI driving as a fallback.** (design, zoom-out, readiness, catalyst, codebase)
- About half of the validated facts are workarounds for the TUI path:
  - `agent_pane_busy` retries;
  - trust-dialog keys;
  - `agent_prompt_stalled` followed by sending Enter;
  - polling `agent_status`;
  - scraping the session JSONL.
- mcorreiam adds more: `agent wait` returns early, and the stall error can be a false negative.
- Headless runs:
  - claude: `claude -p --output-format json`, with the same `--settings`, `--permission-mode dontAsk`, `--tools Read,Grep,Glob`, `--strict-mcp-config` and `--no-chrome`.
  - agy: `agy --print --output-format json`, mentioned unverified in `.egm/refs/mcorreiam-egm-herdr/docs/design.md:156-158`.
  - The dispatcher runs them through `herdr pane run`, with a shell redirect it owns: `… > raw.json; echo $? > exit`.
  - Completion is then process exit, and the report is stdout.
  - Streaming JSON piped through a formatter keeps the pane watchable.
- To verify:
  - `claude -p` works under OAuth (unlike `--bare`, `.egm/feature_concept.md:96`) and skips the trust dialog;
  - `agy --print` honours plan mode and does not block on trust or permission prompts.
- Secondary trade-offs:
  - Headless processes may not trigger herdr's integration hooks, so `agent_status` may not apply.
  - The pane is less interactive.
  - Interactive mode stays necessary for any kind without a headless mode.
- Readiness and zoom-out treat this as the decision that shapes the profile schema, so it should be made before drafting.

### 6. Report collection

**🟡 Reviewers write nothing; the dispatcher writes the report file.** (all seven lenses)
- There is a direct contradiction (comprehension, catalyst):
  - Canonical personas say "Strict Read-Only Protocol … NO tools to … write files" (`UP/agents/comprehension.md:11`).
  - The concept's Goldfish "writes one report" (`.egm/feature_concept.md:64`).
  - The prototype grants Write and `Edit(/$REVIEW/**)` (`.egm/refs/prototype-egm-herd.sh:55,88`), and mcorreiam grants `edit` (`agents/egm-reviewer.md:18-20`). Either way, the dispatch prompt contradicts unforked persona text, and which one wins depends on the model.
  - If agy plan mode blocks all writes, an agy reviewer cannot write a report at all.
- Recommendation:
  - The reviewer's final message is the report. The dispatcher captures it and writes `<run>/…/<reviewer>.md`.
  - The source is a profile field from a closed set: `stdout` (headless), `transcript` (claude JSONL, an undocumented internal path), or `file` (last resort).
  - Test (c) becomes "every write is denied", which is simpler and stronger.
- If transcripts are used, set the session id up front with `claude --session-id <uuid>` instead of reading it back from herdr (codebase).
- Require a terminal marker line, so a partly written report is not counted as done (codebase).

**🟡 Fix the report envelope now.** (comprehension, structural, design, codebase)
- Front matter fields:
  - `schema_version`, run id, dispatch id;
  - reviewer (upstream `name`), persona path and hash;
  - kind, model;
  - stage and round labels (opaque, passed in by the Elephant);
  - target path, hash and range;
  - status (`completed` | `not run`), timestamps, attempt;
  - token usage where available.
- Body: the reviewer's final message, verbatim.
- Do not promise structured findings in v1.

### 7. Completion detection

**🟡 Do not decide "done" from `agent_status` or from what the model replies.** (design, codebase, readiness, comprehension, structural, catalyst)
- herdr: "Antigravity … can be idle while background tasks or subagents keep running, so a settled-state wait is not proof" (`.egm/refs/herdr/herdr-agent-skill.md:59`).
- herdr also documents an `unknown` state that "does not prove completion" (`:55`). The concept's state list leaves it out (`.egm/feature_concept.md:100`).
- **mcorreiam's wait-output trigger is likely broken** (codebase, readiness, design):
  - Its regex `reviewers/[a-z_]+_report\.md *$` (`SKILL.md:85`) matches a line in its own prompt (`SKILL.md:107`).
  - herdr says wait-output "searches the selected snapshot immediately, so output that already exists can match" (`herdr-agent-skill.md:189`).
  - mcorreiam's own `docs/design.md:77-80` contradicts its SKILL.md here.
- A usage-limit screen leaves the agent idle with no report (design).
- Recommendation:
  - Headless: done = exit-status file written by the wrapper shell AND non-empty captured report.
  - TUI: done = settled state AND a per-dispatch nonce sentinel AND the report extracted. The nonce lives in the task file, never in the prompt text or command line.
  - Use `agent_status` for liveness and `blocked` detection only. Treat `unknown` and gone as not done.
  - Outcome classes: `completed`, `not_run:start_failed`, `not_run:blocked`, `not_run:timeout`, `not_run:empty_report`, `not_run:isolation_violation`, `quota_exhausted` (readiness, design).
  - Stop agy reviewers from spawning their own subagents, which defeats idle detection (comprehension).
- Open question (catalyst): whether herdr emits agent-state events to plugin `[[events]]` hooks (`herdr-plugins.mdx:84-86`). If it does, completion could be pushed instead of polled.

### 8. Orchestration: dispatcher API, parallelism, retries, names, cost

**🟡 Split the dispatcher into non-blocking verbs.** (all lenses except structural)
- `dispatch` takes a batch of reviewers, one verbatim prompt file per reviewer, and the run directory. It returns at once with a run id.
- `wait`/`collect --max <below the tool cap>` can be called again safely and prints per-reviewer status and report paths.
- `status` resumes from on-disk state after an Elephant crash.
- The dispatcher, not the Elephant, adds the persona path and the isolation preamble.
- One-reviewer-per-call blocking would either serialize brainstorm's 7 parallel lenses (`UP/skills/egm-brainstorm/SKILL.md:54-56`) or leave the Elephant juggling background jobs.
- Avoid harness-specific background features such as `run_in_background`. They break symmetry between Claude and agy (catalyst).
- Trade-off: the Elephant must obey a polling loop. That is a model-obedience risk, though smaller than obeying a whole method.

**🟡 Retries happen at two layers and multiply.** (design, structural)
- The dispatcher retries only transport errors (pane busy, stalled prompt) inside a time budget.
- Review-level retry ("retry once, then not run") stays with the Elephant, under upstream text.
- The dispatcher never re-runs a review silently and never swaps model or kind (mcorreiam `SKILL.md:171`).
- Never retry quota exhaustion.

**🟡 Token and quota cost is not budgeted.** (design, structural)
- One full planning run dispatches about 23-41 fresh top-level sessions:
  - brainstorm 7;
  - review stage 5 = 2, stage 6 = 1, stage 7 = 3 × 2-5 rounds, stage 8 = 1, stage 9 = 3 × 2-5 rounds (`UP/skills/egm-review-loop/SKILL.md:169-175`);
  - plus retries.
- The success criterion asks for two such runs.
- Seven parallel sessions can hit subscription windows.
- Recommendation:
  - a per-kind `max_parallel` with a queue;
  - record usage from the headless JSON into run state (cheap now, and it feeds the later cost work);
  - consider cheaper default models for structural and style.

**🟡 herdr agent names and platform limits.** (design, codebase, readiness, structural)
- Names must match `[a-z][a-z0-9_-]{0,31}` and be unique among live agents (`.egm/refs/herdr/herdr-agent-skill.md:57`). `egm-implementation-readiness` is already 28 characters.
- Proposal: `<run8>-<alias>-<n>`, with short aliases kept as data and a lookup table in run state.
- The prototype's 6-pane cap (`.egm/refs/prototype-egm-herd.sh:176`) is below brainstorm's 7 lenses.
- The prototype uses `mapfile` (`:101`), which macOS bash 3.2 does not have.
- The prototype runs `rm -rf "$GF_HOME/$2"` on a name taken from front matter, without checking it (`:96`). A name like `..` deletes the parent. Check the name before any path work.
- Language choice: layered config, schema validation and JSONL parsing favour a stdlib-only Python dispatcher over bash + jq + awk (structural, readiness). The draft should decide this.

### 9. Kind profiles and matrix as external data

**🟡 "Profiles are data" will turn into a home-made scripting language unless strategies are bounded.** (design, comprehension, structural, zoom-out, catalyst)
- Trust handling (screen match, then keys), transcript parsing and completion detection are behaviour, not data.
- Three proposals:
  - (a) **Closed strategy set plus one escape hatch** (structural, comprehension, readiness).
    - Profile fields name built-in strategies, for example:
      - `trust: none | screen-confirm{match, keys} | pretrusted`
      - `collect: stdout | claude-jsonl | file`
      - `completion: exit | agent-status | sentinel`
    - The escape hatch is `strategy: exec`, an external hook command whose path is set in user config.
  - (b) **Executable kind adapters** (design). Each adapter is an executable that takes a task file and an output dir and leaves a status file, the captured report and a raw log. `claude` and `agy` ship built in; the company harness is an executable named in user config.
  - (c) **Command templates only** (zoom-out). A command template, an optional settings template and an output parser name. This works if headless passes.
- Trade-off: a new harness that needs a new strategy needs a core change in (a). The owner must accept that, or drop the claim "without touching the core" (`.egm/feature_concept.md:45-46`).
- Proposed profile fields (readiness, structural, codebase):
  - `schema_version`, `kind`, `herdr_kind` (kept separate: the integration is `antigravity-cli`, while mcorreiam uses `--kind agy`);
  - `launch.argv` with placeholders, `mode` (`tui|headless`);
  - `workdir` policy, `instruction_exclusion` (mechanism plus known leaks);
  - `read_allow`, `read_deny`, `write_allow`, `tools`;
  - `non_interactive`, `trust`, `completion`, `collect`;
  - `requires` (minimum versions), `evidence` (smoke-test record).

**🟡 The matrix holds routing only.** (all lenses that read mcorreiam's config)
- Shape: `{schema_version, default: {kind, model}, reviewers: {<upstream frontmatter name>: {kind, model, alias}}}`.
- Keys are checked at load time against the `name:` frontmatter of the resolved upstream `agents/*.md`. Unknown keys are an error; missing keys use `default`.
- No stage lists, no `brainstorm_reviewers`, no fallbacks. Stage makeup comes from upstream at run time.
- `model` is an opaque string per kind, since agy IDs embed effort (`gemini-3.8-flash-medium`).
- `egm-herdr check` fails when a reviewer that the installed skills name is missing from the matrix.

**🟡 Lookup order, merging and trust.** (readiness, structural, comprehension, catalyst)
- Lowest priority first:
  1. repo defaults;
  2. `$XDG_CONFIG_HOME/egm-herdr/` (codebase: or `HERDR_PLUGIN_CONFIG_DIR` when the plugin is installed);
  3. extra directories from an env path, for the private harness;
  4. CLI flags.
- Replace whole items; never deep-merge.
- Ship `egm-herdr config show --resolved`, showing the source file of each value.
- **No profiles loaded from the project under review.** A cloned repo must not be able to inject launch flags such as `--dangerously-skip-permissions`.
- Bind support to evidence (catalyst): a kind is routable only if a smoke stamp exists whose hash covers the profile file and the harness major.minor version. Editing the profile or upgrading the harness invalidates the stamp, which triggers a re-smoke or a loud warning.
  - Trade-off: friction after frequent Claude updates.
- Disagreement: zoom-out and catalyst want two layers only (repo example, user config dir). Readiness and structural add an env override and CLI flags.

### 10. Run state and file layout

**🟡 One stable per-run folder is enough for the later GitHub layer.** (structural, codebase, readiness, zoom-out, catalyst)
- The Elephant passes the run directory explicitly. It resolves `EGM artifact directory:` itself, per `UP/docs/harnesses.md:119-138`. The dispatcher never parses CLAUDE.md or AGENTS.md.
- Proposed layout: `<artifact>/.egm-herdr/runs/<run-id>/run.json`, plus `dispatches/<dispatch-id>/{task.md, raw.*, report.md, status.json, screen.txt?}`.
- `run.json` records:
  - Elephant kind and version;
  - upstream root and version;
  - herdr client and server versions;
  - each harness CLI version;
  - hashes of the resolved matrix and profiles.
- Per-dispatch records:
  - pane id, agent name, session id, cwd;
  - timestamps;
  - every automated keypress with its matched screen text;
  - stall and retry events;
  - report byte count and sha256;
  - final status.
- Before closing a failed pane, dump `herdr agent read --source recent-unwrapped` to the run folder.
- Give `wait` distinct exit codes, so the Elephant can apply upstream's retry rule without guessing.
- Index upstream-owned stage reports but never move or reformat them.
- Publish JSON Schemas for `run.json`, `status.json` and the envelope.
- Zoom-out's caution: do not design schema beyond this in v1.

### 11. Upstream version and the two installed copies

**🟡 Persona paths must come from the Elephant's own upstream copy.** (design, codebase, readiness, comprehension, structural)
- Claude's copy is at `~/.claude/plugins/cache/egm/egm/<ver>`; agy's is at `~/.gemini/config/plugins/egm/` (`UP/docs/harnesses.md:30-38`). They can be different versions.
- Upstream says "Never guess a cache version or search another installed copy" (`UP/docs/claude.md:52-53`).
- Recommendation:
  - The dispatcher uses the root the Elephant's skill resolved, passed in explicitly.
  - Record version and file hashes per run.
  - Warn on a version mismatch.
- Preflight check (structural): the anchor phrases the override depends on (for example "If your harness cannot nest subagents, use direct dispatch") still exist in the installed SKILL.md files. If not, warn about upstream drift.
- An agy reviewer may not be allowed to read the Claude cache path. The review bundle removes that problem (design).
- Disagreement on pinning: comprehension would keep mcorreiam's commit + SHA-256 pin. design, zoom-out, structural and codebase would record the version and hashes but not pin.

### 12. Reuse vs drop

**🟢 From the prototype (`.egm/refs/prototype-egm-herd.sh`).**
- Keep:
  - the `HERDR_ENV`/`jq` preflight (`:39-40`);
  - `state_of` (`:45`), extended for `unknown`;
  - the `agent_pane_busy` retry (`:103-107`);
  - trust handling guarded by an exact screen match (`:108-124`), as profile data;
  - the stall-to-Enter fix (`:146-150`);
  - the short prompt pointing at a task file (`:146`; avoids the macOS line-input limit, `herdr-agent-skill.md:189`);
  - front-matter task files;
  - a fresh cwd and a `--settings` file written to disk;
  - `dontAsk`, `--strict-mcp-config --no-chrome`;
  - the transcript-dir deny (`:61`);
  - closing only panes the tool created (`:222-231`);
  - the exit codes for blocked or missing reports (`:164, 208`).
- Drop or move to user config:
  - the executor role and homelab/kubectl rules (`:14-16, 34, 66-90`);
  - personal deny paths and `$HOME/development` as the read root (`:35, 52-61`);
  - Write/Edit for Goldfish (`:55, 88`);
  - the TSV state, replaced by versioned JSON;
  - the 30-minute blocking stage;
  - the 6-pane grid.
- Fix if reused: `mapfile`, the unchecked `rm -rf`, the per-agent sequential wait.

**🟢 From `mcorreiam/egm-herdr`.**
- Keep (copied text carries its MIT notice in NOTICE):
  - the reviewer prompt clauses: treat the target as data, report an unreadable required file as an incomplete review, do not read other reviewers' reports (`SKILL.md:93-111`);
  - unique report names in a namespace separate from the stage reports;
  - `agent_prompt_stalled` as a possible false negative (`SKILL.md:69`);
  - the "never silently switch model/kind" rule (`SKILL.md:171`);
  - the `herdr integration status` check (`docs/design.md:144-150`);
  - recording installed versions.
- Drop:
  - the rewritten methodology, stages and workflow modes;
  - `review_stages`, `brainstorm_reviewers`;
  - the implementer;
  - fallback and calibration models;
  - the model rationale tables;
  - the OpenCode agents, commands, installer and `check-permissions.py`;
  - the wait-output trigger;
  - asking the user before closing panes.
- Disagreement on pane reuse and the ~80-column split rule (`SKILL.md:38-41`): codebase and comprehension would keep them. design, zoom-out, catalyst and structural would drop them for one tab per stage, closed after collection.
- Zoom-out: if no text is copied verbatim, a README credit is enough and no NOTICE file is needed.

### 13. herdr plugin role

**🟢 Cut `herdr-plugin.toml` and the status pane from v1.** (all seven lenses)
- Plugin actions receive herdr context, not arguments (`.egm/feature_concept.md:101-102`; `herdr-plugins.mdx:265-289`), so the plugin cannot be the dispatch path.
- Install puts the plugin in a herdr-managed folder that changes on update, so the Elephant would still need a stable path.
- `herdr plugin install`/`link` is an owner gate (owner's private brief).
- An `egm-herdr status` command covers the status pane. A status pane can come later as one more reader of the run-state files, like the GitHub layer.
- Keep the `HERDR_PLUGIN_CONFIG_DIR` convention as one possible config home (codebase).

### 14. Terms to define before drafting (comprehension)

**🟡 Several terms are overloaded or undefined.**
- "kind": profile name vs herdr `--kind`, and `agy` vs `antigravity-cli`.
- "profile": no field list.
- "matrix": scope, location, failure behaviour.
- "supported": upstream's end-to-end meaning (`UP/docs/harnesses.md:175-176`) vs the smoke-test meaning. Proposed: "Goldfish-supported (smoke test passed on versions X)" and "Elephant-supported (full run passed)".
- "human gates": EGM's in-skill question points (readiness gate, preset and stage checkpoints, link approval, artifact-folder confirmation, resolve decisions) vs the brief's project gates (repo creation, push, installs).
- "Goldfish-only", "adapter layer" vs "adapter skill" vs "adapter", "dispatcher", "report location", "JEV".
- "delegation override" is used at `.egm/feature_concept.md:63` before it is explained at `:69-76`.
- Recommendation: one glossary line each.

**🟡 Error contract.**
- Every dispatcher failure carries: reviewer, kind, pane id, state, the next command to run, and an exit code (comprehension, codebase).

### 15. Inventive contradictions, patterns and the Ideal Final Result (catalyst, with support from other lenses)

| # | Contradiction | Resolution pattern | Secondary trade-off |
| --- | --- | --- | --- |
| C1 | Follow upstream skill text unchanged vs dispatch somewhere else | Inversion: use upstream's own "cannot nest → direct dispatch" branch; one new row in the Delegation table; the dispatcher copies the Agent tool's contract (prompt in, report out); deny native Agent for `egm:*` | deny-rule syntax unverified; Elephant pays the synthesis context cost |
| C2 | One blocking subagent-like call vs shell timeouts and required parallelism | Segmentation: batch `dispatch` + bounded, repeatable `collect`; dispatcher does the waiting, retry and timeout | two verbs; Elephant must obey a loop; run state becomes required |
| C3 | Reviewer read-only vs reviewer must deliver a report | Extraction: the final message is the report; the dispatcher writes the file (stdout or transcript) | headless is less visible; transcript path is an internal detail |
| C4 | Fresh folder per reviewer vs no trust dialog | Separation in space and time: stable slots trusted once at a setup gate, contents wiped per run; or headless skip; unknown dialog → `not run` | memory carry-over risk; wiping touches another tool's state; pool size caps parallelism |
| C5 | Isolation must be proven vs agy has no verified exclusion or read-deny | Preliminary action (spike first, fallback declared); space segmentation (cwd outside the tree); extraction into the OS (sandbox); cheap copies (review bundle) | sandbox may break OAuth; bundles need path remapping; user `GEMINI.md` stays open |
| C6 | One conversation marker vs a test (a) about instruction files | Segment canaries by channel; behavioural canaries | user-level canaries need a gated write; models may ignore behavioural canaries |
| C7 | Reliable completion vs `agent_status`/wait returning early | Process exit (headless); nonce sentinel the prompt cannot echo (TUI); vote across signals | small models may drop sentinels; more timeout combinations to test |
| C8 | Profiles as data vs data that holds launch argv and can weaken isolation | Self-service reconciliation: an isolation contract the core checks, plus free args; smoke stamp bound to the profile hash and harness version | re-smoke friction after upgrades |

**Ideal Final Result:**
- The Elephant follows upstream EGM text unchanged.
- To the Elephant, a Goldfish looks like a native subagent call that returns the report text.
- The adapter skill is about ten lines of mapping and needs no change across upstream releases unless the Delegation section itself changes.
- The core has no per-kind logic. A kind is a profile file plus a passing smoke stamp, and isolation is enforced outside the agent where possible.
- "Supported" is a machine-checked stamp, not a README claim.
- A reviewer is finished when its process exits, not when its state is inferred.
- Setup happens once, with the human present at a declared gate. No run ever shows a dialog.
- A private company harness is one profile file in the user config dir. Nothing private goes into the public repo.

### 16. YAGNI: what to cut from v1

Reviewers agreed on cutting:
- the herdr plugin manifest and status pane;
- model fallbacks and calibration;
- executor and implementer roles;
- the A/B synthesizer instances and synthesizer emulation (direct dispatch only);
- `--machine` remote panes;
- kinds beyond claude and agy;
- parsing findings into structure;
- cost features;
- routing for bootstrap, lite and drift-report;
- grid-layout tuning.

Proposed by some reviewers, not agreed by all:
- agy as Elephant and the second end-to-end run, moved to v1.1 (zoom-out; design and structural suggest ordering this work last rather than cutting it);
- interactive TUI mode for the v1 kinds, if headless passes;
- SHA pinning;
- pane reuse;
- a declarative profile language (design would use executable adapters instead).

Keep:
- the external config lookup;
- the stable run-state and report schema;
- the canary-based smoke test with saved evidence;
- the per-kind support stamp;
- the `dispatch`/`collect`/`smoke` verbs;
- the `native` kind (if the owner accepts it).

## Goals vs. Non-Goals

**Goals (v1)**
- Run Goldfish reviewers of kind `claude` and `agy`, chosen per reviewer by a matrix, each in its own herdr pane as a fresh session.
- Keep upstream EGM personas, standards and skill text unforked. The adapter changes only where a reviewer runs and how its report comes back.
- Keep kind profiles and the matrix in data loadable from outside the repo. Ship only generic defaults; nothing machine-specific.
- Make isolation a test per kind, with saved evidence. A kind is "supported" only after a real run passes.
- Have no human input during a run except EGM's gates. Reviewers suggest also allowing a documented one-time setup.
- Keep reports and run state as files with a stable structure that a later GitHub layer can read.
- Elephant support in Claude Code and in agy. Reviewers suggest ordering agy-as-Elephant last, or deferring it.

**Non-Goals (v1)**
- Image/video generation and other non-review worker tasks.
- Model routing or decision models (JEV), cost optimisation, local or small models, the company harness.
- GitHub tracking of the EGM process.
- Forking or rewriting EGM methodology, stage lists or personas.
- Using herdr or the prototype to run this project's own EGM process.
- Suggested by reviewers: the herdr plugin manifest and status pane, model fallbacks, executor/implementer roles, synthesizer emulation, remote machines, kinds beyond two.

## Alternatives Weighed

**Rejected by the owner**
- **Fork `mcorreiam/egm-herdr`.** Rejected by author: little reusable code; its Elephant, installer and commands are bound to OpenCode; its history is mostly OpenCode permission patches; its docs are in Portuguese; and its SKILL.md rewrites EGM methodology. (Reviewers confirmed the drift: different stage makeup, a different brainstorm list, wrong reviewer key.)
- **Add a cheap/local-model Goldfish-only kind in v1.** Rejected by author: deferred until the company harness is added.
- **Image/video generation inside this adapter.** Rejected by author: a separate future mechanism.
- **GitHub tracking in v1.** Rejected by author: deferred, but stable file-based reports and state are required.
- **A `native` kind (same-vendor reviewers as the harness's own subagents).** Rejected by author: one dispatch path for every reviewer keeps the Elephant override a single rule instead of a per-reviewer table, gives every reviewer a visible pane and filesystem rules, and makes the "mixed claude + agy" test exercise the mechanism being built.
- **Interactive TUI reviewers driven by keys.** Rejected by author after spikes: headless runs avoid trust dialogs and prompt stalls; lifecycle state is reported by the wrapper instead of screen detection.
- **A dedicated agy HOME to exclude the user-level `GEMINI.md`.** Not pursued by author: agy is accepted at a "measured leak" level, and the shared `GEMINI.md` will be removed on this machine instead.

**Reviewer suggestions (not decisions)**
- **Headless reviewers inside herdr panes instead of driving interactive TUIs** (design, zoom-out, readiness, catalyst, codebase). Completion = process exit, report = stdout, no trust-dialog automation and no reviewer write access. Risk: herdr state hooks may not fire.
- **A `native` kind** (design, codebase, zoom-out, catalyst, comprehension). Same-vendor reviewers use the harness's subagent tool; only foreign kinds go through herdr.
- **No herdr at all for v1** (zoom-out, implied by design and catalyst). The Elephant runs `agy --print` directly through its shell. Rejected inside the reviews themselves as losing visibility and the company-harness path, but named as the minimum baseline.
- **Dispatcher writes reports; reviewers write nothing** (all lenses). This replaces the "reviewer writes its file" model.
- **Override delivered by launcher** (`claude --append-system-prompt-file`) **plus a native-Agent deny rule** (design, catalyst), instead of, or in addition to, an adapter skill as entry point (structural, codebase).
- **Stable pre-trusted slot folders** (catalyst, structural, readiness) instead of a fresh folder per reviewer.
- **Review bundle**: copy the target, persona and standards into the reviewer folder (design, catalyst).
- **OS-level sandbox** (`sandbox-exec`, later bubblewrap) as a harness-independent isolation layer (catalyst).
- **Executable kind adapters** (design) vs **a closed strategy set plus an exec hook** (structural) vs **plain command templates** (zoom-out).
- **Reuse `quaywin/agys`** for agy profile isolation (codebase).
- **Python stdlib dispatcher** instead of bash (structural, readiness).
- **Smallest viable v1** (zoom-out):
  - Claude Code Elephant only.
  - Native claude reviewers plus headless agy reviewers through a small dispatcher.
  - One end-to-end run.
  - agy Elephant in v1.1.

## Key Decisions & Staging for Next Steps

### Decisions already taken by the owner

1. New public repo `setchevest/egm-herdr` under the personal account, not a fork. License Apache-2.0; copied MIT text keeps its notice in a NOTICE file.
2. v1 kinds are `claude` and `agy`, for both roles (Elephant and Goldfish).
3. Kind profiles and the matrix are loadable from outside the repo (for example a user config directory). The private company harness is added later without touching the core and without private details in the public repo.
4. Nothing machine-specific in the repo. Personal deny rules, home paths and the homelab executor role move to local user config; the repo ships generic defaults and examples.
5. Image/video generation and other non-review worker tasks are a separate future mechanism.
6. Out of v1: model routing and decision models (JEV), cost and local models, the company harness, GitHub tracking. Carried forward: reports and run state are files with a stable structure, and the matrix is data an external tool could generate.
7. EGM personas, standards and skill text are read from the installed upstream plugin and not forked.
8. This project is built with standard EGM and native subagents, not with herdr or the prototype.
9. Reviewers run headless (`claude -p`, `agy -p`) inside herdr panes, started with `herdr pane run` through a wrapper. The wrapper reports lifecycle state with `herdr pane report-agent`; the dispatcher waits with `herdr agent wait`; exit code + non-empty captured response decide success. Reviewers write nothing; the dispatcher writes the report file. (Spike evidence: `docs/brainstorm/spikes/2026-11-21-isolation-headless.md`.)
10. Claude Goldfish profile: fresh cwd, `--setting-sources project,local`, `--settings` with read allow/deny rules, `--permission-mode dontAsk`, `--tools Read,Grep,Glob`, `--strict-mcp-config --no-chrome`.
11. agy Goldfish is accepted at a "measured leak" support level: `--mode plan`, `--settings` with `allowNonWorkspaceAccess: false`, exact-file read allows and explicit denies (`.env`, transcript stores, `write_file(/)`, `command(*)`, `mcp(*)`, `read_url(*)`, `execute_url(*)`). Known leaks, documented: user-level `GEMINI.md`, web search and subagents not fully controllable. The profile must keep every read inside the allow list, because a soft-deny empties the report. An empty response with `denied_actions` is `not run:empty_report`, never "no findings".
12. One dispatch path: no `native` kind. Every Goldfish goes through herdr, whatever the Elephant's harness.
13. One override file, two entry points. The Elephant override (never use the native subagent tool for EGM reviewers, never dispatch the synthesizer, dispatch every reviewer through `egm-herdr`, synthesize in the Elephant's own context per upstream's direct-dispatch branch) lives in one file. A launcher (`egm-herdr elephant claude|agy`) starts a dedicated Elephant session with that file injected and, for Claude, a session-scoped `--settings` that denies the native subagent tool. An in-session command reads the same file, then the upstream skill by path, with no enforcement. The v1 success criterion is tested from the launcher only; the in-session entry is experimental until its own obedience run passes. The dispatcher keeps one record per reviewer so a missing dispatch is visible.
14. Defaults carried into the draft unless the owner changes them: a documented one-time setup outside "the run" is allowed (adapter on `PATH`, EGM plugin installed in agy, each an owner gate when it writes into another tool's config); a kind's smoke test passes at 3 of 3 runs.

### Open questions to resolve before `egm-draft` (most blocking first)

1. **What happens if agy Goldfish fails the isolation smoke test?** The v1 criterion cannot be met as written if agy cannot deny secret reads, exclude user `GEMINI.md`, or deny writes. Choose before the spike:
   - two support levels (isolated / measured leak);
   - `allow_unsupported` opt-in;
   - claude-only Goldfish in v1;
   - or an OS sandbox or review bundle as the enforcement layer.
   - *Owner decision (2026-11-21):* accept agy Goldfish as it is, at a documented "measured leak" support level (user-level `GEMINI.md` loads; web search and subagents stay available). The owner will remove the shared `~/.gemini/GEMINI.md` on this machine later; the adapter does not depend on that.
   - *Spike result:* agy fails (a) on the user-level `GEMINI.md` and cannot disable web search or subagents; (b) and (c) pass with explicit denies. A review bundle (target, persona and standards copied into the reviewer's cwd, where reads are allowed by default) would also avoid the soft-deny trap. A dedicated agy HOME with a one-time login is the only untested route to (a).
2. **How reviewers run: headless or interactive TUI?** This decides the report source, completion detection, trust handling and the profile schema. It depends on spikes: `claude -p` under OAuth, `agy --print` with plan mode, and whether either fires the herdr state hooks.
   - *Spike result and owner decision:* headless for both kinds inside herdr panes, with no dialogs. The wrapper reports lifecycle state with `herdr pane report-agent`, and the dispatcher waits with `herdr agent wait`. Exit code + non-empty response confirm success.
3. **Override shape, delivery and enforcement.**
   - Confirm the mapping: subagent tool = `egm-herdr dispatch`; this harness cannot nest; always direct dispatch; never dispatch the synthesizer; it replaces `docs/claude.md` Reviewer isolation.
   - Choose delivery: adapter skill as entry point, or launcher with a system-prompt append.
   - Choose enforcement: native Agent deny rule or not.
   - Define the obedience check and how many runs count.
   - *Owner decision (2026-11-21):* see Key Decision 13 (one override file; launcher with enforcement is the tested entry; in-session command is experimental). Still open: whether agy can deny `invoke_subagent` (not found in the spikes), so agy enforcement may be obedience-only; the obedience spike (spike 3) has not run. EGM is not installed in agy on this machine (`agy plugin list`: none); installing it is an owner gate.
4. **Is a `native` kind allowed, and does a native claude reviewer count toward "a mix of claude and agy"?**
   - *Owner decision (2026-11-21):* no. One path: every Goldfish, including claude reviewers under a Claude Elephant, goes through herdr.
5. **Success-criterion scope.**
   - Keep agy-as-Elephant and the second end-to-end run in v1, or move them to v1.1?
   - Allow a documented one-time setup gate (shell allow rules, trust of slot folders, plugin read rules) outside "the run"?
   - Is 3-of-3 smoke runs per kind the threshold?
   - *Default carried forward:* see Key Decision 14 (one-time setup allowed; 3 of 3). agy-as-Elephant and both end-to-end runs stay in v1 (owner decision).
6. **Claude isolation definition.**
   - Does test (a) require excluding user-level `~/.claude/CLAUDE.md` and user hooks?
   - If yes, is losing the herdr state hook acceptable (`--setting-sources`, `CLAUDE_CONFIG_DIR`, or `claude --agent egm:egm-*`)?
   - *Spike result:* `--setting-sources project,local` excludes user settings and keeps OAuth. The herdr hook is not needed in headless mode, so losing it costs nothing. A planted user `CLAUDE.md` test is still pending (owner gate).
7. **Report contract.** Confirm that reviewers write nothing and the dispatcher writes `report.md` with a fixed front-matter envelope, so test (c) becomes "all writes denied".
8. **Dispatcher API and timeouts.**
   - Batch `dispatch` + repeatable `collect`/`wait` + `status`.
   - Per-reviewer and per-stage timeouts.
   - `max_parallel`.
   - Outcome classes and exit codes.
   - Which layer retries what.
9. **Kind extensibility model.** Closed strategy set plus exec hook, executable adapters, or command templates? Does adding a new collection strategy count as "touching the core"?
10. **Config format, lookup order and safety.**
    - JSON + JSON Schema with `schema_version`.
    - Two layers or four.
    - No project-level profiles.
    - Whether support is bound to a smoke stamp hashed over profile and harness version.
11. **Run-state location and layout** (for example `<artifact>/.egm-herdr/runs/<id>/…`) and how much schema to publish in v1.
12. **Upstream resolution.** Use the Elephant's own copy, record version and hashes, an anchor-phrase drift check; pin by SHA or not.
13. **Trust policy.** Stable pre-trusted slots vs fresh folders with logged key automation; written exception to herdr's "ask the user" guidance.
14. **Implementation language** (bash 3.2-safe vs Python stdlib), agent naming scheme, pane layout (tab per stage vs pane reuse) and cleanup policy.
15. **NOTICE file.** Only if mcorreiam text is copied verbatim; otherwise a README credit.

### Spikes reviewers want before `egm-draft` (highest risk first)

1. **agy isolation.** Project and user `GEMINI.md`/`AGENTS.md` canaries; `.env` read via Read, Grep and Glob; a write attempt under `--mode plan`; reads of the persona from outside the workspace; transcript-store reads; whether a per-run config or profile dir exists without breaking OAuth.
2. **Headless mode in herdr panes.** `claude -p --output-format json` under OAuth (trust dialog? permission rules hold?), `agy --print --output-format json` (trust and permission prompts? exit code? full report on stdout?), and whether herdr state hooks fire.
3. **Override obedience.** A stub dispatcher that logs calls and returns canned reports. Run brainstorm and one review-loop stage with a Claude Elephant, then an agy Elephant. Pass = zero native subagent calls, correct dispatch count and names, no synthesizer, gates kept, 3 of 3.
4. **Unattended start per kind.** Slot pre-trust vs key automation vs headless; ten consecutive cold starts with no input; agy permission prompts.
5. **Completion and report source per kind.** agy transcript location, the Claude JSONL path rule, idle-while-background, a sentinel the prompt echo cannot match.
6. **Elephant shell constraints.** Long-command timeout and allowlist syntax in Claude Code and agy, with no prompts across a full stage.
7. **Claude user-level exclusion.** Plain `claude` vs `claude --agent egm:egm-*` vs `--setting-sources`, with a planted user-level canary and hook.
8. **Mechanics.** Agent-name scheme, more than 6 parallel panes, language choice.
