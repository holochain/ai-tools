# Holochain 0.6 → 0.7 API change reference

Complete catalog of the breaking changes, organized by which layer of the app they hit.
The official guide lives at
`https://developer.holochain.org/resources/upgrade/upgrade-holochain-0.7/` — if anything
here disagrees with the published guide or with `docs.rs` at the project's pinned version,
those win.

## Contents

- [Toolchain and dependency versions](#toolchain-and-dependency-versions)
- [The action model rewrite (the big one)](#the-action-model-rewrite)
- [FlatOp renames](#flatop-renames)
- [Validation callback signatures](#validation-callback-signatures)
- [Coordinator zome changes](#coordinator-zome-changes)
- [Removed HDK functions](#removed-hdk-functions)
- [Rust client / test changes](#rust-client--test-changes)
- [JavaScript client changes](#javascript-client-changes)
- [Conductor config changes](#conductor-config-changes)
- [Subtle runtime changes](#subtle-runtime-changes)

## Toolchain and dependency versions

| What | 0.6 | 0.7 |
|---|---|---|
| holonix flake ref | `main-0.6` | `main-0.7` |
| Node.js (in flake) | `nodejs_20`/`nodejs_22` | `nodejs_24` (guide) — `nodejs_22` also works; `nodejs_20` fails nix eval on the 0.7 nixpkgs pin |
| `hdi` | 0.7.x | **0.8.0** |
| `hdk` | 0.6.x | **0.7.0** |
| `holochain` (dev-dep) | 0.6.x | **0.7.0** |
| `@holochain/client` | ^0.20.x | **^0.21.0** |
| `@holochain/hc-spin` | 0.600.x | **^0.700.0** |

`holochain` crate feature renames:

- `sqlite-encrypted` → `encryption`
- `wasmer_sys` → `wasmer-sys-cranelift`
- `transport-iroh` → removed (iroh is now the only transport; the feature flag is gone)

After bumping the flake ref: `nix flake update && git add flake.* && nix develop`.

## The action model rewrite

In 0.6 every `Action` enum variant repeated the common fields (`author`, `timestamp`,
`action_seq`, `prev_action`). In 0.7 an action is a `header` (the common fields) plus a
`data` enum (`ActionData`) holding the variant-specific fields.

Type renames:

| 0.6 | 0.7 |
|---|---|
| `Action::Create(c)` (matched on the action) | `ActionData::Create(c)` (matched on `action.data`) |
| `Create` / `Update` / `Delete` | `CreateData` / `UpdateData` / `DeleteData` |
| `CreateLink` / `DeleteLink` | `CreateLinkData` / `DeleteLinkData` |
| `EntryCreationAction` | `TypedAction<EntryCreationData>` |

Accessor changes — common fields are now methods (or live under `.header`):

```rust
// 0.6
action.author
action.timestamp

// 0.7
action.author()
action.timestamp()
```

## FlatOp renames

| 0.6 | 0.7 |
|---|---|
| `FlatOp::StoreEntry` | `FlatOp::CreateEntry` |
| `FlatOp::StoreRecord` | `FlatOp::CreateRecord` |
| `FlatOp::RegisterUpdate` | `FlatOp::Update` |
| `FlatOp::RegisterDelete` | `FlatOp::Delete` |
| `FlatOp::RegisterCreateLink` | `FlatOp::Link(OpLink::CreateLink)` |
| `FlatOp::RegisterDeleteLink` | `FlatOp::Link(OpLink::DeleteLink)` |
| `FlatOp::RegisterAgentActivity` | `FlatOp::AgentActivity` |

## Validation callback signatures

Entry validation takes `TypedAction<...>` instead of the old concrete action types.
(This is the guide's typed style; real upgrades have also used plain `Action`
parameters with a runtime `matches!` guard — both compile against hdi 0.8. See
`zome-migration.md` for the trade-off.)

```rust
// 0.6
pub fn validate_create_post(
    _action: EntryCreationAction,
    _post: Post,
) -> ExternResult<ValidateCallbackResult>

// 0.7
pub fn validate_create_post(
    _action: TypedAction<EntryCreationData>,
    _post: Post,
) -> ExternResult<ValidateCallbackResult>
```

Link validation collapses to a single parameter — base, target, and tag now live on the
action's `data`:

```rust
// 0.6
pub fn validate_create_link_all_posts(
    _action: CreateLink,
    _base_address: AnyLinkableHash,
    target_address: AnyLinkableHash,
    _tag: LinkTag,
) -> ExternResult<ValidateCallbackResult>

// 0.7
pub fn validate_create_link_all_posts(
    action: TypedAction<CreateLinkData>,
) -> ExternResult<ValidateCallbackResult> {
    let action_hash = action.data.target_address.into_action_hash()?;
    // base is action.data.base_address, tag is action.data.tag
```

## Coordinator zome changes

**`signal_action` / any code matching on actions** — match on `.data`:

```rust
// 0.7
match &action.hashed.content.clone().data {
    ActionData::CreateLink(create_link) => { /* ... */ }
    ActionData::DeleteLink(delete_link) => { /* ... */ }
    // ...
}
```

**`Record::new()`** takes `RecordEntry`, not `Option<Entry>`:

```rust
// 0.6: Record::new(action, Some(entry))
Record::new(action, RecordEntry::Present(entry))
```

**`get_agent_activity()`** gained a fourth parameter and the return type was renamed
`AgentActivity` → `AgentActivityStatus`:

```rust
let activity: AgentActivityStatus = get_agent_activity(
    agent,
    ChainQueryFilter::new(),
    ActivityRequest::Full,
    GetOptions::default(), // new required parameter
)?;
```

It can now also return `ChainStatus::Closed`.

**`ChainFilter`** — builder methods replaced by direct constructors:

```rust
// 0.6: ChainFilter::new(chain_top).until_hash(oldest_hash)
let filter = ChainFilter::until_hash(chain_top, oldest_hash);

// 0.6: ChainFilter::new(chain_top).take(10)
let filter = ChainFilter::take(chain_top, 10);
```

**`must_get_agent_activity`** — new response variants your code may need to handle:
`UntilHashMissing`, `UntilHashAfterChainHead`, `UntilTimestampIndeterminate`,
`IncompleteChain`. Also: a `ChainFilter` with `LimitConditions::Take(0)` now returns an
error instead of an empty result.

## Removed HDK functions

`block_agent()` and `unblock_agent()` are gone — agent blocking is now system-level via
warrants. If a zome called them, the calls must be removed (there is no drop-in
replacement at the zome level).

## Rust client / test changes

- `dump_network_stats` on both admin and app clients now returns
  `HolochainTransportStats` (previously the two returned inconsistent types).
- Signing traits moved to `holochain_keystore`:

  ```rust
  // 0.6: use holochain_types::prelude::SignedActionHashedExt;
  use holochain_keystore::SignedActionHashedExt;
  ```

  Also there: `ValidationReceiptExt`, `WarrantOpExt`, `ReportEntryFetchedOpsExt`.

## JavaScript client changes

**`SignedActionHashed` is no longer generic**, and the variant-specific action types
(`Create`, `CreateLink`, `Delete`, `DeleteLink`, …) are no longer exported:

```typescript
// 0.6
import { Create, SignedActionHashed } from "@holochain/client";
action: SignedActionHashed<Create>;

// 0.7
import { SignedActionHashed } from "@holochain/client";
action: SignedActionHashed;
```

**Common action fields moved under `header`:**

```typescript
// 0.6: action.hashed.content.author
action.hashed.content.header.author;
action.hashed.content.header.timestamp;
```

**`dumpNetworkStats`** returns a unified `ApiTransportStats`; per-connection stats are
nested under `transport_stats` and `is_webrtc` is renamed `is_direct`:

```typescript
const stats: ApiTransportStats = await client.dumpNetworkStats();
const connected = stats.transport_stats.connections.length;
const direct = stats.transport_stats.connections.filter((c) => c.is_direct).length;
```

**`ConnectionServices`**: `signalingServerUrl` → `relayServerUrl`.

## Conductor config changes

- Removed: `signal_url`, `webrtc_config`, `chc_url` (tx5/WebRTC transport is gone; iroh
  relay replaces the signal server)
- Moved: `request_timeout_s` now lives inside the `network` section
- Renamed: `db_sync_strategy` → `db_sync_level`, with new values `Full` / `Normal` /
  `Off` (replacing `Fast` / `Resilient`)
- New optional: `wasm_backend` — `"cranelift"`, `"LLVM"`, or `"wasmi"`
- Local iroh relay without TLS needs:

  ```yaml
  network:
    advanced:
      irohTransport:
        relayAllowPlainText: true
  ```

## Subtle runtime changes

- **No data migration path.** DNA hashes change, databases were renamed, and 0.7 cannot
  read 0.6 databases. Every dev environment needs `hc sandbox clean`; shipped apps need
  their conductor data cleared or a fresh data dir.
- WASM compiled modules are now cached in the database — no `wasm-cache` directory.
- App/web-app manifests **reject unknown fields** (previously ignored). Stale keys in
  `happ.yaml` / `web-happ.yaml` that 0.6 tolerated are now hard errors.
- `hc sandbox` network types: only `mem` and `quic` (`webrtc` removed).
- `StorageInfo`: `authored_data_size` / `cache_data_size` removed; source-chain data is
  counted in `dht_data_size`.
- DNA migration support via the new `InitProperties` type.
- `ListApps` can filter for `AppStatusFilter::AwaitingMemproofs`.
