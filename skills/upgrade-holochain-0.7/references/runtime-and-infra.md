# Runtime, conductor config, manifests, and toolchain for 0.7

Everything outside the zome/client source: nix, manifests, conductor config, transport
infrastructure, and the build-toolchain fights. Mostly relevant to launchers/runtimes
(moss-class projects) and CI, but the manifest and nix sections hit every project.

## Contents

- [Nix flake and Node version](#nix-flake-and-node-version)
- [Manifest schema is strict now](#manifest-schema-is-strict-now)
- [Conductor config changes](#conductor-config-changes)
- [Transport: signal server is gone, iroh relay replaces it](#transport-signal-server-is-gone-iroh-relay-replaces-it)
- [No data migration — the clean break](#no-data-migration--the-clean-break)
- [Build toolchain fights](#build-toolchain-fights)
- [rc vs final drift](#rc-vs-final-drift)

## Nix flake and Node version

```nix
holonix.url = "github:holochain/holonix?ref=main-0.7";
```

then `nix flake update && git add flake.* && nix develop`. The holonix bump drags
nixpkgs to `nixos-26.05`, where **`nodejs_20` is flagged EOL/insecure and fails devshell
evaluation**. Move to `nodejs_22` or `nodejs_24` (the guide says 24; real upgrades used
both — either works, 24 is the forward-looking choice). What actually matters:

- Pin CI's Node to the **same major** as the flake — native modules (bufferutil,
  utf-8-validate) are built against the running Node ABI.
- Expected lockfile movement: holochain `0.7.0`, lair `v0.7.1`, kitsune2 `v0.5.0`,
  scaffolding `v0.700.0-rc.0`.

`holochain` crate features if you depend on it directly: `sqlite-encrypted` →
`encryption`, `wasmer_sys` → `wasmer-sys-cranelift`, `transport-iroh` → gone (iroh is
the only transport now). The plain release binaries from
`github.com/holochain/holochain/releases` already include the default feature set —
no special "iroh variant" to fetch.

## Manifest schema is strict now

0.7 **rejects unknown fields** in happ/web-happ manifests where 0.6 silently ignored
them — with no `manifest_version` bump to warn you. Two real examples of `happ.yaml`
role fixes:

```yaml
# stale 0.6-era keys that now hard-error — delete them:
    dna:
      path: ../dnas/group/workdir/group.dna
      # properties: ~        <- remove
      # network_seed: ~      <- remove
      # version: ~           <- remove

# or the older flat form that must move to `modifiers`:
    dna:
      path: ./syn-test.dna
      # was: properties: ~ / uuid: ~ / version: ~
      modifiers:
        network_seed: ~
        properties: ~
      installed_hash: ~
```

`dna.yaml` files did **not** need changes in any of the reference upgrades.

Launcher-level consequence: changing manifest schema changes the packed happ bytes, so
**sha256 checks against previously-published happs fail** after unpack/repack. Moss's
fix: fall back to a legacy-hash computation
(`rustUtils.legacyHappSha256FromBytes`) when the primary hash mismatches, and save the
file under the *expected* hash so later lookups find it.

## Conductor config changes

Removed / moved / renamed:

| field | change |
|---|---|
| `network.signal_url` | removed (no signal server) |
| `network.webrtc_config` | removed (no WebRTC/tx5) |
| `chc_url` | removed |
| `device_seed_lair_tag`, `danger_generate_throwaway_device_seed`, `dpki` | removed |
| top-level `request_timeout_s` | moved to `network.request_timeout_s` |
| `db_sync_strategy: Fast\|Resilient` | `db_sync_level: Full\|Normal\|Off` — a **semantic** change, not a rename; `Resilient` maps to `Normal` |
| `network.base64_auth_material` | split into `base64_auth_material_bootstrap` / `_relay` |
| `wasm_backend` | new optional: `"cranelift"` / `"LLVM"` / `"wasmi"` |
| `db_max_readers`, `incoming_request_concurrency_limit` | new, fine at defaults |

A launcher migrating existing on-disk configs should delete the removed keys and map
the renamed ones (moss's `holochainManager.ts` does exactly the delete-list above).

Dev-mode plaintext relay (local, non-TLS):

```yaml
network:
  advanced:
    irohTransport:
      relayAllowPlainText: true
```

(replaces 0.6's `tx5Transport: { signalAllowPlainText: true }` — and make sure the
`advanced` object is actually assigned back onto the config after mutation).

## Transport: signal server is gone, iroh relay replaces it

The tx5/WebRTC transport and its signal server are removed. The conductor needs a
`bootstrap_url` and a `relay_url`. Operational facts from deploying this:

- `kitsune2-bootstrap-srv` (0.4.0-dev.7+) serves **both** roles from one node: the
  bootstrap at `/`, the iroh relay at `/relay`. One local process yields both URLs in
  dev mode.
- `kitsune2-bootstrap-srv --production` binds both `0.0.0.0:443` and `[::]:443`; on
  Linux with default `bindv6only=0` the second bind collides (`EADDRINUSE`) and the
  process **stays up with no listener** (the error is discarded internally). Pass an
  explicit `--listen [::]:443` instead.
- CLI/launcher surfaces that accepted a signaling URL (`--signaling-url`, ICE-server
  options) should be removed — ICE/webrtc options are dead config on 0.7.

## No data migration — the clean break

0.7 cannot read 0.6 databases: DNA hashes change (0.6 and 0.7 agents form disjoint
networks), databases were renamed, and the WASM store layout changed (compiled modules
now live in the database; no `wasm-cache` directory). Per project type:

- **Dev environments:** `hc sandbox clean`. Also note `hc sandbox` network types are
  now only `mem` and `quic` (`webrtc` removed).
- **Launchers/runtimes:** isolate the 0.7 profile directory from the 0.6 one rather
  than migrating — e.g. moss bumped its appId and `binariesAppendix`
  (`moss-0.15` → `moss-0.16`) so fresh installs land in a fresh dir. Decide and state
  explicitly that this is a fresh-start release.
- `StorageInfo` consumers: `authored_data_size` / `cache_data_size` are gone;
  source-chain data now counts inside `dht_data_size`.

## Build toolchain fights

Seen in real upgrades; keep these in the back pocket rather than debugging from
scratch:

- **Undefined HDI host symbols at link time** (`__hc__must_get_entry_1`,
  `__hc__dna_info_2`): newer rustc rejects undefined symbols on
  `wasm32-unknown-unknown`. Fix: follow the holonix-provided Rust toolchain (or pin
  rustc to what the holochain release was built with) rather than an ad-hoc newer one.
- **arm64 rust-lld**: may need `--import-undefined` passed to wasm-ld.
- Running `wasm-opt` over zomes changes the DnaHash — fine during a version jump that
  already breaks the hash, but be aware it's part of the hash.
- Node-version pins live in three places that must agree: `flake.nix`, CI workflow
  files, local dev.

## rc vs final drift

The 0.7 rc line and 0.7.0 final differ in real API surface (e.g.
`OpActivity::CreateAgent` gained an `agent` field only in final). Rules:

- Pin final versions (`=0.7.0` / `=0.8.0` style pins are reasonable in a runtime).
- When a local holochain checkout disagrees with the crates.io pin, **the cargo
  registry source is authoritative** for what your code compiles against.
- Write tolerant patterns (`{ action, .. }`) where an rc/final field difference bit.
