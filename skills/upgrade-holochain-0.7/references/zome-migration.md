# Migrating zomes (Rust) from 0.6 to 0.7

The mechanical recipe for integrity and coordinator zomes, with the patterns that worked
in real upgrades (presence, moss, syn). Bump deps first (`hdi = "0.8.0"`,
`hdk = "0.7.0"`, `holochain_serialized_bytes = "0.0.57"` — note the skew: hdi 0.7→0.8
but hdk 0.6→0.7), then let the compiler walk you through the list below. Expect a wall
of errors on the first build; they fall into a small number of mechanical categories.

## Contents

- [Integrity: FlatOp variant renames](#integrity-flatop-variant-renames)
- [Integrity: link ops became a nested match](#integrity-link-ops-became-a-nested-match)
- [Integrity: EntryCreationAction is gone](#integrity-entrycreationaction-is-gone)
- [Integrity: common fields are methods now](#integrity-common-fields-are-methods-now)
- [Integrity: other renames and field moves](#integrity-other-renames-and-field-moves)
- [Coordinator: signal_action and friends](#coordinator-signal_action-and-friends)
- [Coordinator: other API changes](#coordinator-other-api-changes)
- [What did NOT change](#what-did-not-change)

## Integrity: FlatOp variant renames

| 0.6 | 0.7 | Note |
|---|---|---|
| `FlatOp::StoreEntry(e)` | `FlatOp::CreateEntry(e)` | |
| `FlatOp::StoreRecord(r)` | `FlatOp::CreateRecord(r)` | |
| `FlatOp::RegisterUpdate { .. }` | `FlatOp::Update(_)` | struct variant → **tuple** variant |
| `FlatOp::RegisterDelete { .. }` | `FlatOp::Delete(_)` | struct variant → **tuple** variant |
| `FlatOp::RegisterAgentActivity { .. }` | `FlatOp::AgentActivity(_)` | struct variant → **tuple** variant |
| `FlatOp::RegisterCreateLink { .. }` | `FlatOp::Link(OpLink::CreateLink { .. })` | see next section |
| `FlatOp::RegisterDeleteLink { .. }` | `FlatOp::Link(OpLink::DeleteLink { .. })` | see next section |

Renaming alone is not enough for the middle three — the pattern syntax changes from
`{ .. }` to `(_)`. And because two variants merged into `Link`, an exhaustive match
(no wildcard arm) must be restructured, not just renamed.

Naming collision to keep straight: `FlatOp::CreateEntry`, `OpRecord::CreateEntry`, and
`OpEntry::CreateEntry` are three different types. Only the outer `FlatOp` wrapper names
changed; the inner `OpRecord`/`OpEntry` variant shapes are mostly intact.

## Integrity: link ops became a nested match

In 0.6, `RegisterCreateLink` destructured `base_address` / `target_address` / `tag`
right at the match site. In 0.7 those live on the action's data:

```rust
FlatOp::Link(op_link) => match op_link {
    OpLink::CreateLink { link_type, action } => {
        // action: TypedAction<CreateLinkData>
        // base/target/tag are on action.data (Deref lets `action.base_address` work too)
    }
    OpLink::DeleteLink { original_action, link_type, action } => {
        // action: TypedAction<DeleteLinkData>; base is on action.data,
        // target/tag are on original_action.data
    }
},
```

Two proven ways to port a 0.6 zome that had a big `match link_type` under
`RegisterCreateLink`:

**Extract a helper taking the old bindings** (syn's pattern — keeps the link-type match
body byte-identical):

```rust
fn validate_create_link(
    link_type: LinkTypes,
    base_address: AnyLinkableHash,
    target_address: AnyLinkableHash,
) -> ExternResult<ValidateCallbackResult> {
    match link_type { /* unchanged 0.6 body */ }
}
// at the match site:
OpLink::CreateLink { link_type, action } => validate_create_link(
    link_type,
    action.base_address.clone(),
    action.target_address.clone(),
),
```

**Clone into locals before converting the action** (presence's pattern — needed when the
per-link-type validators take the action too):

```rust
OpLink::CreateLink { link_type, action } => {
    let base_address = action.data.base_address.clone();
    let target_address = action.data.target_address.clone();
    let tag = action.data.tag.clone();
    match link_type {
        LinkTypes::AllAgents =>
            validate_create_link_all_agents(action.into(), base_address, target_address, tag),
        // ...
    }
}
```

Order matters in the second pattern: `action.into()` (via
`impl From<TypedAction<D>> for Action`) **moves** the action, so pull the fields out
first. This borrow-order dance is most of the line churn in a typical integrity port.

## Integrity: EntryCreationAction is gone

`EntryCreationAction` no longer exists. Two replacement styles, both correct:

**Style A — `TypedAction<EntryCreationData>` (what the upgrade guide shows, preserves
type safety).** hdi 0.8 ships `TypedAction::<EntryCreationData>::try_from_action(action)`
explicitly designed for a validate callback's `?`-chain, plus a `TryFrom<Action>` impl:

```rust
pub fn validate_create_post(
    _action: TypedAction<EntryCreationData>,
    _post: Post,
) -> ExternResult<ValidateCallbackResult>
```

**Style B — plain `Action` plus a runtime guard (what presence and moss shipped —
a simpler mechanical port, at the cost of type-level guarantees):**

```rust
pub fn validate_update_room_info(
    _action: Action,
    _room_info: RoomInfo,
    _original_action: Action,          // was EntryCreationAction
    _original_room_info: RoomInfo,
) -> ExternResult<ValidateCallbackResult>

// where 0.6 had EntryCreationAction::try_from(original_action):
if !matches!(original_action.data, ActionData::Create(_) | ActionData::Update(_)) {
    return Ok(ValidateCallbackResult::Invalid(
        "Original action for an update must be a Create or Update action".to_string()));
}
```

With Style B, call sites that wrapped (`EntryCreationAction::Create(action)`) become
`action.into()`. Prefer Style A for new code; use Style B when you want the smallest
diff. Don't mix styles within one zome.

## Integrity: common fields are methods now

`Action` is now `{ header: ActionHeader, data: ActionData }`. Common fields moved to the
header and are exposed as methods returning references:

```rust
// 0.6                          // 0.7
action.author                   action.author()        // returns &AgentPubKey — compare with &x
action.timestamp                action.timestamp()
action.action_seq               action.action_seq()
action.prev_action              action.prev_action()   // returns Option<&ActionHash>!
```

`prev_action()` returning an `Option` forces a new error path that 0.6 didn't have
(e.g. in `OpActivity::CreateAgent` handling):

```rust
let prev_action_hash = action.prev_action().cloned().ok_or(wasm_error!(
    WasmErrorInner::Guest("CreateAgent action must have a previous action".to_string())))?;
```

Trap: `TypedAction<D>` `Deref`s to its `data`, so `action.base_address` compiles while
the adjacent `action.author` does not (that one is `action.header.author` or the
method). The inconsistency makes compile errors look random — it isn't; field = data,
method/header = common fields.

## Integrity: other renames and field moves

- **Variant payload structs renamed** with a `Data` suffix: `Create`→`CreateData`,
  `Update`→`UpdateData`, `Delete`→`DeleteData`, `CreateLink`→`CreateLinkData`,
  `DeleteLink`→`DeleteLinkData`, `AgentValidationPkg { membrane_proof, .. }` →
  `AgentValidationPkgData { membrane_proof }`.
- **Field projections move through `.data`:**
  ```rust
  // 0.6: action.original_action_address / delete.action.deletes_address
  action.data.original_action_address.clone()
  delete_entry.action.data.deletes_address.clone()
  ```
- **`entry_type()` returns `Option<&EntryType>`** (was `&EntryType` on
  `EntryCreationAction`): patterns become `Some(EntryType::App(t)) =>` and `t` usually
  needs a clone. Entry visibility moved: read `app_entry_type.visibility`, not
  `action.entry_type().visibility()`.
- **`OpRecord` variants dropped redundant hash fields** — read them off the action:
  ```rust
  // 0.6: OpRecord::UpdateEntry { original_action_hash, .. }
  OpRecord::UpdateEntry { app_entry, action } =>
      must_get_valid_record(action.data.original_action_address.clone())?;
  // 0.6: OpRecord::DeleteEntry { original_action_hash, .. }
  OpRecord::DeleteEntry { action } =>
      must_get_valid_record(action.data.deletes_address.clone())?;
  // 0.6: OpRecord::DeleteLink { original_action_hash, base_address, action }
  OpRecord::DeleteLink { action } => {
      let base_address = action.data.base_address.clone();
      must_get_valid_record(action.data.link_add_address.clone())?;
  }
  ```
- **`OpActivity::CreateAgent` is `{ action, agent }`** in 0.7.0 final (the rc line
  lacked `agent`). Write the tolerant form `OpActivity::CreateAgent { action, .. }` if
  you don't need the key.
- **`debug!` is feature-gated out of the hdi 0.8 integrity prelude.** A leftover
  `debug!` in `genesis_self_check` or validation fails to build; delete the call rather
  than importing something.
- **Nice new helper:** `action.into_entry_data()` returns `Option<(EntryHash, EntryType)>`,
  collapsing the old two-arm `Action::Create(c) | Action::Update(u)` entry-type matches.

## Coordinator: signal_action and friends

Match on `.data`; variants are `ActionData::*`:

```rust
// on an owned/cloned action (action is moved into emit_signal later — clone):
match action.hashed.content.data.clone() {
    ActionData::CreateLink(create_link) => { /* ... */ }
    ActionData::DeleteLink(delete_link) => { /* ... */ }
    // ...
}
// on a fetched record (record.action() returns &Action — borrow, don't move):
match &record.action().data {
    ActionData::CreateLink(create_link) => { /* ... */ }
    // ...
}
```

The naive edit `match record.action().data` gives "cannot move out of a shared
reference" — the `.data` projection changed the borrow story relative to 0.6's
direct enum match.

Inside the arms, `create_link.zome_index` / `create_link.link_type` still work (those
fields are on `CreateLinkData`), but `create_link.author` / `.timestamp` /
`.action_seq` / `.prev_action` are **gone** — they're on
`action.hashed.content.header` now. Zomes that only touched variant-specific fields
port with a pure rename; zomes reading common fields off the variant need real edits.

## Coordinator: other API changes

- `Record::new(action, Some(entry))` → `Record::new(action, RecordEntry::Present(entry))`.
- `get_agent_activity` takes a fourth argument and the return type is renamed:
  ```rust
  let activity: AgentActivityStatus = get_agent_activity(
      agent, ChainQueryFilter::new(), ActivityRequest::Full, GetOptions::default())?;
  ```
  It can now return `ChainStatus::Closed`.
- `ChainFilter` builders became constructors: `ChainFilter::until_hash(top, hash)`,
  `ChainFilter::take(top, n)` (was `ChainFilter::new(top).until_hash(..)` / `.take(..)`).
- `must_get_agent_activity` has new response variants to handle: `UntilHashMissing`,
  `UntilHashAfterChainHead`, `UntilTimestampIndeterminate`, `IncompleteChain`; and
  `LimitConditions::Take(0)` is now an error, not an empty result.
- `block_agent()` / `unblock_agent()` removed with no zome-level replacement (blocking
  is system-level via warrants). Delete the calls.
- In Rust client/test code: signing traits moved —
  `use holochain_keystore::SignedActionHashedExt;` (was in
  `holochain_types::prelude`). Also there: `ValidationReceiptExt`, `WarrantOpExt`.

## What did NOT change

Verified untouched across three real upgrades — don't spend time looking for problems
here:

`#[hdk_entry_types]` / `#[unit_enum]` / `#[hdk_link_types]` / `#[hdk_extern]` /
`#[hdk_extern(infallible)]`, `genesis_self_check(GenesisSelfCheckData)`,
`validate_agent_joining`, `EntryTypes::deserialize_from_type`, `LinkTypes::from_type`,
`must_get_valid_record` / `must_get_entry` / `must_get_action`, `wasm_error!` /
`WasmErrorInner::Guest`, `into_entry_hash()` / `into_action_hash()` /
`into_any_dht_hash()`, `LinkQuery::try_new` + `get_links`, `GetOptions::local()` /
`GetOptions::network()`, `get_details` / `Details::Record`, `DeleteLinkInput::new`,
`ChainTopOrdering`, `create_cap_grant` / `CapGrantEntry::new` /
`GrantedFunctions::Listed`, `emit_signal` / `send_remote_signal` /
`recv_remote_signal` / `call_remote`, `post_commit(Vec<SignedActionHashed>)`,
`agent_info()` / `call_info()` / `zome_info()`, and `dna.yaml` manifests.

A coordinator `Signal` enum carrying untyped `SignedActionHashed` keeps working
unchanged on the wire — the JS side sees the new `{header, data}` content shape
automatically.
