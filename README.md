# egm-herdr

Run [Elephant-Goldfish Model (EGM)](https://github.com/google/elephant-goldfish-model) Goldfish
reviewers as fresh agent sessions in [herdr](https://github.com/herdrdev/herdr) panes, across
harnesses and models. EGM's personas, standards and skills stay canonical and unforked; this
adapter only changes *where* a Goldfish runs and *how* its report comes back.

**Status:** brainstorm closed, design not drafted, no code yet. Tracking issue:
[#1](https://github.com/setchevest/egm-herdr/issues/1).

## v1 scope

- Elephant in Claude Code or the Antigravity CLI (`agy`), started with a launcher.
- Goldfish reviewers of kind `claude` and `agy`, chosen per reviewer by a matrix, each run
  headless in its own herdr pane.
- Isolation is a test per kind, with saved evidence. `claude` reviewers pass it; `agy`
  reviewers run at a documented "measured leak" level.

Out of v1: other harnesses, local or cheap models, decision models, GitHub tracking of the EGM
process, non-review worker tasks.

## Documents

- [Concepts brief](docs/brainstorm/concepts_brief.md): the brainstorm output, with Goldfish
  findings, spike results, owner decisions and open questions.
- [Spike evidence, 2026-11-21](docs/brainstorm/spikes/2026-11-21-isolation-headless.md):
  isolation of `claude -p` and `agy -p` reviewers, and headless runs in herdr panes.
- [Feature concept](docs/brainstorm/feature_concept.md): the input given to the Goldfish
  reviewers.

## Credits

- [google/elephant-goldfish-model](https://github.com/google/elephant-goldfish-model)
  (Apache-2.0): the method, personas and standards this adapter runs.
- [mcorreiam/egm-herdr](https://github.com/mcorreiam/egm-herdr) (MIT): an earlier EGM × herdr
  adapter for OpenCode and agy. Ideas taken from it: the reviewer → {kind, model} matrix, the
  reviewer prompt rules, and several herdr dispatch findings. No code or text is copied yet; if
  any is, its MIT notice goes into a `NOTICE` file.

## License

[Apache-2.0](LICENSE).
