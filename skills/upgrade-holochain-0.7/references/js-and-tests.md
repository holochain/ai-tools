# Migrating JS/TS clients and tests from 0.6 to 0.7

The `@holochain/client` 0.21 changes, the ecosystem package map, and the traps that
survived typechecking in real upgrades.

## Contents

- [Version map](#version-map)
- [Keeping one copy of @holochain/client](#keeping-one-copy-of-holochainclient)
- [Action shape: fields moved under header](#action-shape-fields-moved-under-header)
- [SignedActionHashed lost its generic](#signedactionhashed-lost-its-generic)
- [The ActionData type hole in client 0.21](#the-actiondata-type-hole-in-client-021)
- [Tests: tryorama moved to a fork](#tests-tryorama-moved-to-a-fork)
- [The grep sweep (tests are not typechecked)](#the-grep-sweep-tests-are-not-typechecked)
- [Network stats and debugging tooling](#network-stats-and-debugging-tooling)
- [Staged migration is legitimate](#staged-migration-is-legitimate)

## Version map

| package | 0.6 line | 0.7 line |
|---|---|---|
| `@holochain/client` | `^0.20.x` | `^0.21.0` |
| `@holochain/hc-spin` | `^0.600.x` | `^0.700.0` |
| `@holochain-open-dev/utils` / `stores` / `elements` / `profiles` | `^0.601.x` | `^0.700.0` |
| `@holochain/tryorama` | `^0.19.x` | **dead end** — see below |
| `@holochain-open-dev/tryorama` | — | `^0.20.0` |
| `@theweave/api` etc. | 0.5.x/0.6.x | 0.7.x |
| `@msgpack/msgpack` | ^2.7 | ^2.8 |

Three versioning schemes coexist, so don't pattern-match one bump onto another:
holochain-open-dev packages encode the conductor version (`0.601.x`→`0.700.0`), the
client is on its own line (`0.20`→`0.21`), and tryorama is on yet another
(`0.19`→`0.20`).

`hc-spin` must jump the full major or it cannot install a 0.7 happ.

The Rust `hc_zome_profiles_*` git deps: use **tag `v0.700.0`** — the `main-0.7` branch
was not advanced to the release. And beware: those crates *compile* against a dual
hdi-0.6+0.7 dependency graph but then produce 0.6-targeted WASM that will not load in a
0.7 conductor. Compiling is not proof the dependency actually moved; check `Cargo.lock`
for exactly one `hdi`/`hdk` version.

## Keeping one copy of @holochain/client

If any dependency pins a different client version, npm/yarn install two copies, and
because the interop types are structural you get absurd errors like
"`AgentPubKey` is not assignable to `AgentPubKey`", plus `HoloHashMap`/`instanceof`
checks silently failing at runtime. Two proven fixes:

```json
// npm: root package.json
"overrides": { "@holochain/client": "^0.21.0", "@holochain-open-dev/utils": "^0.700.0" }
// yarn: root package.json
"resolutions": { "@holochain/client": "^0.21.0" }
```

These blocks are load-bearing — carry them forward with bumped versions, don't
"clean them up". During the rc period several `@holochain-open-dev/*` packages
hard-pinned exact client versions, which is exactly what these overrides fight. Once
every dependency requests `^0.21.0` the pin can be dropped (moss did, after tryorama
0.20.0 final shipped).

Yarn-1 extra: `file:` tarball deps are cached by version — after repacking a patched
client run `yarn cache clean @holochain/client` before `yarn install`.

## Action shape: fields moved under header

Common action fields now live under `header`; variant fields under `data`:

```typescript
// 0.6                                   // 0.7
action.hashed.content.author             action.hashed.content.header.author
action.hashed.content.timestamp          action.hashed.content.header.timestamp
action.hashed.content.type               action.hashed.content.data.type
```

For `EntryRecord` from `@holochain-open-dev/utils`: `record.action.author` →
`record.action.header.author`, `record.action.timestamp` →
`record.action.header.timestamp`. `EntryRecord` still converts the timestamp from
microseconds to milliseconds for you — but only on `header.timestamp`. Raw
`Record` paths (`signed_action.hashed.content.header.timestamp`) still carry
microseconds; divide before `new Date()`.

The `hashed` / `signature` wrapper structure is unchanged — `signal.action.hashed.hash`
etc. keep working; only `hashed.content` gained the header/data split.

## SignedActionHashed lost its generic

```typescript
// 0.6
import { Create, CreateLink, SignedActionHashed } from "@holochain/client";
action: SignedActionHashed<Create>;

// 0.7
import { SignedActionHashed } from "@holochain/client";
action: SignedActionHashed;
```

The variant-specific action types are no longer exported from the client. Code that
used the generic for narrowing now needs either a runtime check on
`action.hashed.content.data.type` or the typed hierarchy from
`@holochain-open-dev/utils` (below). Expect a few `as unknown as SignedActionHashed`
casts at boundaries where old generic types flowed through app-level interfaces.

## The ActionData type hole in client 0.21

`@holochain/client` 0.21's `ActionData` union **omits `Create` and `Update`** — the
vanilla `Record` type literally cannot describe an app-entry record. This is a client
type bug, not yours. Symptoms: "Type `SignedTypedActionHashed<AnyActionData>` is not
assignable to type `SignedActionHashed`" and similar.

`@holochain-open-dev/utils` ships the workaround hierarchy: `AnyActionData`
(= `ActionData | Create | Update`), `TypedAction<D>`, `SignedTypedActionHashed<D>`,
`AnyRecord`, and `EntryRecord.record` is typed `AnyRecord`. Passing a client `Record`
*into* these is fine (narrower → wider); passing an `AnyRecord` back into an interface
declared with the client's `Record` needs a cast (`record as Record`) or — cleaner —
declare the interface with `AnyRecord` in the first place.

## Tests: tryorama moved to a fork

Upstream `@holochain/tryorama` tops out at client ^0.20 (Holochain 0.6) and is a dead
end for 0.7. The maintained 0.7-compatible line is the fork
**`@holochain-open-dev/tryorama` `^0.20.0`** — a package swap, not a version bump:

```typescript
import { runScenario, dhtSync, PlayerApp } from '@holochain-open-dev/tryorama';
```

Call-site API is essentially unchanged (`addPlayersWithApps`, `shareAllAgents`,
`callZome`, `dhtSync`). Shapes worth knowing in 0.7-era tests:

```typescript
const appBundleSource: AppBundleSource = { type: 'path', value: happPath }; // tagged form
player.appWs.on('signal', (signal) => {
  if (signal.type === SignalType.App) signals.push(signal.value.payload);   // tagged union
});
```

Use `^0.20.0` final, not the rc — the final carries the v2 DhtOp `dhtSync` fix (rc
versions can hang or mis-report sync on 0.7).

## The grep sweep (tests are not typechecked)

vitest/esbuild transpile without typechecking, so 0.6-shaped field accesses in test
helpers survive the upgrade silently and only explode at runtime (e.g.
`encodeHashToBase64(undefined)`, `new Date(NaN)`). This produced a real shipped bug in
one of the three reference upgrades. After the mechanical migration, sweep the whole
repo — including `tests/`:

```bash
# high signal — header fields, on action-shaped access paths
grep -rnE '\.(action|content)\.(author|timestamp|action_seq|prev_action)\b' --include='*.ts' .
grep -rnE 'hashed\.content\.(author|timestamp|action_seq|prev_action|type)\b' --include='*.ts' .
grep -rn 'SignedActionHashed<' --include='*.ts' .

# broad backstop — variant fields that moved under .data
# (noisy: entry_type/tag/link_type collide with app-level names — triage, don't assume)
grep -rnE '\.(entry_type|entry_hash|original_action_address|original_entry_address|deletes_address|deletes_entry_address|base_address|target_address|link_add_address|zome_index|link_type|tag)\b' --include='*.ts' .
```

Every hit from the first group should either go through `.header.` / `.data.` or be
justified; the second group needs triage rather than blanket edits. The data-field list
is the complete set from the `*Data` structs in `holochain_integrity_types` 0.7.0.

## Network stats and debugging tooling

- `dumpNetworkStats()` returns `ApiTransportStats`; connection data is nested under
  `transport_stats`, and `is_webrtc` → `is_direct`:
  ```typescript
  const stats: ApiTransportStats = await client.dumpNetworkStats();
  const direct = stats.transport_stats.connections.filter((c) => c.is_direct).length;
  ```
- `ConnectionServices.signalingServerUrl` → `relayServerUrl`.
- State-dump consumers: `ChainOp` tuple values reshaped from
  `[signature, Action, entry]` to `[SignedAction, RecordEntry]` (and `CreateEntry` ops
  carry only a signature first) — the action is `tuple[0].data`, the entry is
  `tuple[1].Present`. `WarrantOp` content: `.timestamp` → `.data.timestamp`.
  `SourceChainJsonRecord`: `record.action.type` → `record.action.data.type`.

## Staged migration is legitimate

You do not have to move DNA and UI in one step. A proven staging: upgrade the DNA +
tests first, defer the UI (still on client ^0.20) to a follow-up. npm workspaces
tolerate the two client majors coexisting in one tree — the 0.21 copies live in
`tests/node_modules` while the root hoists 0.20 for the UI. What you cannot do is mix
the two majors **within one package**, and the UI cannot talk to a 0.7 conductor until
it moves. If you stage, record the deferred step somewhere discoverable (CLAUDE.md,
README) so it isn't forgotten.
