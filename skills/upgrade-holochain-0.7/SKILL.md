---
name: upgrade-holochain-0.7
description: Use this skill when upgrading any Holochain app, library, or runtime from Holochain 0.6 to 0.7 — or when debugging code that was just upgraded. Trigger on any request mentioning "upgrade to 0.7", "holochain 0.7", "hdk 0.7" / "hdi 0.8", "@holochain/client 0.21", holonix main-0.7, or on compile/type errors mentioning ActionData, TypedAction, EntryCreationAction, FlatOp variants (StoreEntry/RegisterUpdate/RegisterCreateLink), SignedActionHashed generics, "no field author on type Action", db_sync_strategy, or signal_url. Also trigger when a dependency bump to hdk 0.7/hdi 0.8/client 0.21 produces a wall of errors, even if the user doesn't say "upgrade". Encodes the official upgrade guide plus field-tested knowledge from three real upgrades (an app DNA, a client library, and an Electron runtime), including gotchas the guide doesn't cover.
---

# upgrade-holochain-0.7

You are upgrading a Holochain project from 0.6 to 0.7. The official guide is at
`https://developer.holochain.org/resources/upgrade/upgrade-holochain-0.7/` — this skill
encodes it plus what three real upgrades (presence: app DNA + tryorama tests; syn:
zome + TS client-library stack; moss: Electron runtime + conductor management) actually
hit in practice. Where this file disagrees with `docs.rs` at the project's pinned
version, docs.rs wins.

The upgrade is 90% mechanical. The compiler drives most of the Rust work; the danger is
concentrated in three places: **untypechecked test/TS code** (breaks at runtime, not
build time), **dependency-graph traps** (things that compile but don't work), and
**infrastructure** (transport, manifests, databases). Work the checklist in order and
don't skip the verification sweep at the end.

## Big picture — what changed in 0.7

1. **The action model was rewritten.** `Action` is now a struct
   `{ header, data }`: common fields (`author`, `timestamp`, `action_seq`,
   `prev_action`) live in `header`; the old enum variants became `ActionData` with
   `*Data`-suffixed payload structs. This ripples through every integrity zome,
   `signal_action`, and every JS field access.
2. **tx5/WebRTC transport and the signal server are gone.** iroh is the only
   transport; a relay URL replaces the signaling URL; conductor config changed shape.
3. **No data migration path.** 0.7 can't read 0.6 databases and DNA hashes change —
   0.6 and 0.7 agents form disjoint networks. Every environment needs
   `hc sandbox clean` or a fresh profile dir.

## Step 0 — inventory before editing

Classify the project (zome-only library / full happ with UI and tests / launcher-
runtime) and grep to learn which change classes even apply. In each of the three
reference upgrades, whole sections of the guide were no-ops — knowing that up front
saves the time otherwise spent hunting for problems that aren't there.

```bash
# Rust side — which zome changes apply?
grep -rn 'EntryCreationAction\|FlatOp::\|signal_action\|get_agent_activity\|ChainFilter\|block_agent\|Record::new' --include='*.rs' zomes/ dnas/ crates/ 2>/dev/null
# JS side — which client changes apply?
grep -rn 'SignedActionHashed<\|hashed\.content\.\|\.action\.author\|\.action\.timestamp\|dumpNetworkStats\|signalingServerUrl\|tryorama' --include='*.ts' -r . --exclude-dir=node_modules 2>/dev/null
# Infra — conductor config / manifests / signal server?
grep -rn 'signal_url\|webrtc\|db_sync_strategy\|signaling' --include='*.yaml' --include='*.yml' --include='*.ts' -r . --exclude-dir=node_modules 2>/dev/null
```

Read the matching reference file before editing that layer:

- `references/api-changes.md` — the complete guide-derived change catalog (all layers)
- `references/zome-migration.md` — Rust integrity/coordinator recipe with proven patterns
- `references/js-and-tests.md` — client 0.21, tryorama fork, type holes, grep sweep
- `references/runtime-and-infra.md` — nix, manifests, conductor config, relay/bootstrap, toolchain

## Version cheat sheet

| what | 0.7 value | trap |
|---|---|---|
| holonix | `github:holochain/holonix?ref=main-0.7` | drags nixpkgs to 26.05 → `nodejs_20` fails eval; use `nodejs_22`/`_24` and match CI |
| `hdi` | **0.8.0** | version skew: hdi 0.7→0.8 … |
| `hdk` | **0.7.0** | … but hdk 0.6→0.7. Bumping only one gives trait-resolution walls, not clean errors |
| `holochain` (dev-dep) | 0.7.0 | features: `sqlite-encrypted`→`encryption`, `wasmer_sys`→`wasmer-sys-cranelift`, `transport-iroh` gone |
| `holochain_serialized_bytes` | 0.0.57 | mismatch = "two versions of SerializedBytes" type-identity errors |
| `@holochain/client` | ^0.21.0 | enforce ONE copy via `overrides`/`resolutions` |
| `@holochain/hc-spin` | ^0.700.0 | older cannot install 0.7 happs |
| `@holochain-open-dev/*` | ^0.700.0 | scheme changed from 0.601.x |
| tryorama | `@holochain-open-dev/tryorama` ^0.20.0 | **package swap** — upstream `@holochain/tryorama` is a 0.6 dead end |
| profiles zome (Rust git dep) | tag `v0.700.0` | `main-0.7` branch was not advanced to release |

## Upgrade order (compile gate after each step)

1. **Toolchain**: flake ref → `main-0.7`, Node version, `nix flake update`, enter shell.
2. **Rust deps**: hdi/hdk/holochain/holochain_serialized_bytes bumps; git zome deps to
   their 0.7 tags. Then check `Cargo.lock` has exactly one hdi/hdk version — a dual
   0.6+0.7 graph can still compile and silently produce 0.6-targeted WASM that a 0.7
   conductor won't load.
3. **Integrity zomes** (`references/zome-migration.md`): FlatOp renames (some struct→
   tuple variants), the `Link(OpLink::…)` nested match, `EntryCreationAction`
   replacement (pick ONE style: `TypedAction<EntryCreationData>` or plain `Action` +
   `matches!` guard), field→method accessors, `entry_type()` now `Option`.
4. **Coordinator zomes**: `signal_action` matches on `.data` as `ActionData::*` (mind
   the clone-vs-borrow asymmetry), `Record::new` takes `RecordEntry`,
   `get_agent_activity`/`ChainFilter` signature changes, delete `block_agent` calls.
5. **Manifests**: 0.7 rejects unknown fields in `happ.yaml`/`web-happ.yaml` — delete
   stale keys or move to the `modifiers:` form. `dna.yaml` usually needs nothing.
6. **Tests**: swap to `@holochain-open-dev/tryorama ^0.20.0`; client bump in
   `tests/package.json`.
7. **UI / client TS** (`references/js-and-tests.md`): client ^0.21.0, `.header.`/
   `.data.` field moves, de-generic `SignedActionHashed`. May be legitimately deferred
   to a follow-up — npm workspaces tolerate both client majors in one tree, just not in
   one package. If deferred, record it in the repo.
8. **Runtime/conductor** (launchers only, `references/runtime-and-infra.md`): config
   field migration, signal-server removal, bootstrap+relay URLs, fresh profile dir.
9. **Clean state**: `hc sandbox clean` (or equivalent) everywhere, then run the full
   test suite.

## The traps, ranked by cost

1. **TS code that isn't typechecked** (vitest/esbuild test helpers, debug paths):
   0.6-shaped accesses like `record.action.timestamp` survive the build and throw at
   runtime (`encodeHashToBase64(undefined)`, `Invalid time value`). This shipped a real
   bug in one reference upgrade. Antidote: the grep sweep below, applied to **all** TS,
   not just `src/`.
2. **Two copies of `@holochain/client`** → "`AgentPubKey` is not assignable to
   `AgentPubKey`", instanceof checks silently false. Keep the `overrides`/`resolutions`
   block; it is load-bearing.
3. **Dual hdi graph → 0.6 WASM.** See step 2 above.
4. **client 0.21's `ActionData` union omits `Create`/`Update`** — the vanilla `Record`
   type can't describe an app-entry record. Use `AnyRecord`/`AnyAction` from
   `@holochain-open-dev/utils`, or a cast at the boundary. Client type bug, not yours.
5. **`TypedAction` Deref illusion**: `action.base_address` works (deref to data) while
   the adjacent `action.author` doesn't (it's `action.author()` / `.header.author`).
   Field = variant data; method/header = common fields.
6. **Borrow-order in link arms**: `action.into()` moves the action — clone
   base/target/tag out of `action.data` first.
7. **`Resilient` → `Normal`**: `db_sync_strategy`→`db_sync_level` is a semantic
   remap, not a rename.


## Final verification sweep

After everything compiles and before declaring done:

```bash
# stale 0.6 action-shape accesses (runtime bombs in untypechecked code):
grep -rn '\.action\.author\b\|\.action\.timestamp\b\|hashed\.content\.\(author\|timestamp\|type\)\b\|SignedActionHashed<' \
  --include='*.ts' -r . --exclude-dir=node_modules
# stale Rust patterns that can hide in cfg'd-out or unbuilt code:
grep -rn 'EntryCreationAction\|FlatOp::Store\|FlatOp::Register\|block_agent' --include='*.rs' -r . --exclude-dir=target
# dead transport config:
grep -rn 'signal_url\|signaling\|webrtc\|iceServers' -r . --exclude-dir=node_modules --exclude-dir=target
# exactly one hdi/hdk in the lockfile:
grep -A1 '^name = "hd[ik]"$' Cargo.lock
```

Then: `hc sandbox clean`, run zome tests AND the UI against a real conductor (the
staged-migration case means a green Rust build proves nothing about the UI), and if the
project publishes happs, re-verify sha256 handling (manifest schema change alters
packed bytes — see `references/runtime-and-infra.md`).

Announce clearly in your summary: the DNA hash changed, 0.6 and 0.7 peers are disjoint
networks, and every user/dev environment needs its conductor state cleared or a fresh
profile. This is by design (no migration path), but it must never be a surprise.

## Scope notes

- This skill covers 0.6 → 0.7 only. A project on 0.5 must first absorb the 0.5→0.6
  changes (`LinkQuery`, `GetOptions::local()/network()`, `DeleteLinkInput`, …) — see
  the 0.6 upgrade guide.
- For general Holochain development practice (DNA-hash tripwires, docs.rs
  verification, test strategy), defer to the `holochain-dev` skill; this skill is only
  the migration.
- rc-line and 0.7.0-final APIs differ in places (e.g. `OpActivity::CreateAgent` gained
  an `agent` field in final). Pin finals; the cargo registry source is authoritative
  over any local holochain checkout.
