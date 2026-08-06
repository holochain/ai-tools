# ai-tools

A collection of AI-agent assets (Claude Code skills, eventually plugins and MCP servers) for working with the Holochain ecosystem.

## What's here

### [skills/holochain-dev](skills/holochain-dev/)

A Claude Code skill that encodes semantic Holochain knowledge — the parts that LLMs get wrong because their training data contains obsolete alpha-version APIs, and the parts that are genuinely non-obvious even when the APIs are current.

**What it does** (not an exhaustive list):

- Detects when a proposed change affects the DNA hash and, on protected branches, pauses and offers the dev-branch escape hatch rather than silently making the edit
- Verifies HDK/HDI API claims against `docs.rs/hdk/<project-version>` rather than recall (current target: HDK `0.6.1-rc.5` / HDI `0.7.1-rc.5`)
- Inverts the msgpack-version-pin reflex when debugging Rust↔TS serialization errors
- Knows the AgentPubKey-as-EntryHash trick for O(1) joining-timestamp lookups
- Generates sweettest (not tryorama) tests and bootstraps the harness from scratch if the project doesn't have one
- Wraps `cargo` and test commands with `nix develop -c` when `flake.nix` is present
- Detects existing project idioms (coordinator return shape, serialization strategy, signal placement, single-vs-multi-DNA, etc.) and stays consistent rather than imposing a style

**Installation:** copy `skills/holochain-dev/` into `~/.claude/skills/` or wherever your Claude Code setup looks for skills. Eventual plan is to ship as an installable plugin.

### [skills/upgrade-holochain-0.7](skills/upgrade-holochain-0.7/)

A Claude Code skill for upgrading Holochain apps, libraries, and runtimes from 0.6 to 0.7. Encodes the [official upgrade guide](https://developer.holochain.org/resources/upgrade/upgrade-holochain-0.7/) plus field-tested knowledge from three real upgrades (presence: app DNA + tryorama tests; syn: zome + TS client-library stack; moss: Electron runtime + conductor management):

- An ordered upgrade workflow with compile gates and a step-0 inventory that identifies which change classes apply before editing
- The complete API-change catalog (action model rewrite, FlatOp renames, validation signatures, client 0.21, conductor config)
- Proven migration patterns for the awkward spots: the `Link(OpLink::…)` nested match, `EntryCreationAction` replacement styles, clone-vs-borrow in `signal_action`
- The gotchas the guide doesn't cover: the `@holochain-open-dev/tryorama` package swap, the client 0.21 `ActionData` type hole, dual-hdi graphs producing 0.6 WASM, untypechecked test code hiding runtime bombs, manifest schema strictness
- A final grep sweep that catches what the compiler can't

Intended as the companion the upgrade guide can point AI-assisted developers to.

### [notes/](notes/)

Research and design documents from building the skill:

- `holochain-skill-planning.md` — original handoff notes
- `research-app-patterns.md` — pattern survey of unyt-app, dino-adventure, emergence
- `research-mewsfeed-friction.md` — Claude friction patterns mined from mewsfeed session history
- `research-vines-acorn-patterns.md` — orthogonal pattern data from vines and acorn
- `skill-design-v1.md` — synthesis and design brief, with resolved decisions and iteration-2 corrections

These are preserved so future iterations of the skill have the context that shaped it.

## Status

Experimental. First deliverable (`holochain-dev` skill) has been eval-tested through two iterations against five representative Holochain dev prompts, with a measurable pass-rate lift over vanilla Claude Code on the tests where the skill is load-bearing (DNA tripwire, scratch-space semantics, host-fn cost model).

## Aspiration

If the skill proves useful in practice, donate to the `holochain` GitHub org so it can be maintained alongside the HDK.
