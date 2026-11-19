> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Mail delegate core: request handling, storage, consent, crypto (key: delegate-core)

Reference aliases (all paths are relative to the repo root):
`D/` = `delegates/mail-delegate/`, `H` = `D/src/handlers.rs`, `S` = `D/src/store.rs`,
`C` = `D/src/consent.rs`, `E` = `D/src/env.rs`, `K` = `D/src/crypto.rs`, `L` = `D/src/lib.rs`,
`T` = `D/src/tests.rs`, `TF`/`TR`/`TS`/`T2`/`T3` = `D/src/tests/{faults,recovery,resilience,round2,round3}.rs`,
`P` = `protocol/src/lib.rs`. `H:123` means line 123 of that file.

## 1. Purpose and place in the system

One WASM delegate (`mail-delegate` 0.3.0, edition 2024, `D/Cargo.toml:1-7`) that holds every
identity's seeds, contacts and local mail state, and signs/decrypts for the page. It replaces the
v0.2.x identity and local-state delegates. It is outside the workspace with its own lockfile
(`Cargo.toml:47-50` root exclude; `D/Cargo.toml:9-17`). Its delegate key is
`blake3(blake3(wasm) || [])`: parameters are empty (`ui/src/mail_delegate/node.rs:28-32`) and
`process` ignores them (`L:49`). The files mapped here are the request side: gate, dispatch,
identities, items, v1 crypto, exports, consent, storage and the host abstraction. The background
jobs (`jobs.rs`, `watch.rs`, `inbox_jobs.rs`) and inbox v2 (`inbox_v2.rs`, `sealing.rs`) share
`Space`, `put_item` and the entry point but are other maps.

Two hosts run the same code: a node (FFI exported by `#[delegate]` under the
`freenet-main-delegate` feature, `D/Cargo.toml:59-61`) and the offline UI build, which links the
crate as an rlib and calls `handlers::process` over a page-local `Env`
(`ui/src/mail_delegate/local.rs:62-99,129-131`).

## 2. Walkthrough

### `D/Cargo.toml`
Every WASM-reaching dependency is pinned `=x.y.z` except `ed25519-dalek = "2.1.1"` (reason at
`:24-30`; pinned in practice by the lockfile and `--locked`). `freenet-stdlib =0.12.1`,
`freenet-migrate =0.8.0` (only `ExportedSecrets` is used, `H:1741`), `ml-dsa =0.1.1`,
`ml-kem =0.3.3`, `rand_core =0.10.1`. No `getrandom`/`rand` (`:14-16`). Features: `default = []`,
`freenet-main-delegate`, `v1-transition` (`:55-64`). `crate-type = ["cdylib","rlib"]` (`:53`).
Dev-deps: the inbox v2 contract crate and `ghostkey-common` (`:44-50`).
`[profile.dev.package."*"] opt-level = 2` (`:69-70`) because the sweeps generate PQ keys.

### `L` (lib.rs)
Module list (`L:19-28`). Manifest `lifecycle=[NodeStarted]`, `capabilities=[Background]`,
`wakeups=[rescan=900, v1watch=3600, renew=21600, clock=86400]` (`L:41-45`); tags must equal
`jobs::Tick::ALL` (`D/src/jobs.rs:82-87`). stdlib allows at most 4 wake-ups, so the manifest is
full. `process` forwards to `handlers::process` (`L:47-54`).

### `H` (handlers.rs)
**Entry, `process` (`H:34-93`).** Classification:

| Inbound | Gate | Action |
|---|---|---|
| `ApplicationMessage` | `check_origin`, then refuse `processed` (`H:41-46`) | decode `RequestEnvelope`, `handle_request` (`H:49-51`) |
| `UserResponse` | `check_origin` (`H:54`) | `consent::answer` |
| `Lifecycle(NodeStarted)` | none | `background(.., "node-started")` (`H:62-64`) |
| `WakeupFired{tag}` | none | known tag runs, unknown tag returns `[]` (`H:65-73`) |
| `ContractNotification`, `GetContractResponse` | none; id must be 32 bytes | run with `Trigger::Contract`; payload never read (`H:74-81`) |
| `SubscribeContractResponse`, `UpdateContractResponse` | none | ignored, `Ok([])` (`H:85-88`) |
| anything else (`PutContractResponse`, `DelegateMessage`, `UnsubscribeContractResponse`, `Lifecycle(Installed)`) | | `Err(DelegateError::Other)` (`H:89-91`) |

`background` runs `watch::run` and logs `run_line` through `Env::log` (`H:97-110`).

**Origin gate, `check_origin` (`H:123-147`).** Served: only `Some(MessageOrigin::WebApp(id))` with
a 32-byte id that is not all zero (`watch::GLOBAL`, `D/src/watch.rs:48`). `Delegate(_)`, `None`
and any future variant get `Err(DelegateError::Other(..))`: no protocol response, nothing read or
written. The 32 bytes become `Origin` and name the namespace.

**`handle_request` (`H:167-306`).** Wrong `version` gives `UnsupportedVersion`; an `op` that does
not deserialize gives `UnknownOperation` (also what a malformed known op gets) (`H:172-195`).
`respond` wraps one `Response` as `ApplicationMessage.processed(true)` (`H:149-165`).
`RESPONSE_HEADROOM = 16 KiB` (`H:31`).

| Op | Handler | Success reply | Notes |
|---|---|---|---|
| `GetStatus` | `status` `H:439` | `Status` | `identities` = index length, incl. deleting/unreadable (`H:463`); usage is the record, 0 if unknown (`S:371`) |
| `ListIdentities` | `H:570` | `Identities` | skips deleting and unusable records (`H:421-437`) |
| `ListDeletingIdentities` | `H:548` | `Identities` | first runs `prune_recordless_identities` (`H:485-546`), errors ignored |
| `CreateIdentity` | `H:669` | `CreatedIdentity{identity, recovery_key}` | zero entropy is `BadInput`; alias 1-64 non-control chars, description up to 1024 (`H:356-371`); alias clash is `Conflict{0}`; cap 16 |
| `ImportIdentity` | `H:746` | `Imported{identity, already_present}` | never overwrites a usable record; re-indexes an unindexed one; rewrites a listed unreadable/corrupt one (`H:793-832`); refuses one being deleted |
| `UpdateIdentity` / `ConfirmIdentity` | `H:835` | `Ack{generation}` | CAS; a retry of an already-applied change is acked (`H:849-853`) |
| `ListItems` | `H:1310` | `Items{items, more}` | limit clamped 1..=256 (`H:1308,1326`), byte budget 512K-16K, cursor `after` is exclusive id order; prunes recordless entries that have a `c`/`f`/`d` marker |
| `GetItem` | `H:1136` | `Item(Option)` | `Unreadable` if listed but absent with no marker |
| `PutItem` | `H:1152` | `Ack{generation}` | body up to 256 KiB (`P:30`); `None` = create-only, `Some(g)` = CAS, same-body retry acked (`H:1185-1191`) |
| `DeleteItem` | `H:1247` | `Deleted` | absent/unreadable record is deleted without a generation check (`H:1255-1265`); Kept/Archived/Flags leave a tombstone first (`H:1292-1306`) |
| `DecryptV1` | `H:1385` | `Decrypted(Vec<Result>)` | up to 32 ciphertexts, per-entry `BadInput`/`CapReached` |
| `SealV1Message` | `H:1408` | `SealedV1{content, signature, sender_vk}` | signs only the ciphertext it just made |
| `SignV1Authority` | `H:1447` | `Signature` | delegate supplies its own ek |
| `SignV1Removal` | `H:1526` | `Signature` | 1-1024 strictly ascending hashes, all present in the locally held v1 inbox; payload must not start with `STATE_UPDATE` or `fmail/` |
| `V1AftParams` | `H:1587` | `AftParams(json)` | only with `v1-transition`; else `Refused` (`H:1597-1600`). **Contains the ML-DSA seed, no prompt** |
| `RunJobs` | `H:324` | `JobsRun` plus contract requests | notes the origin in `ix/bg/origins` if it has identities |
| `RevokeExportGrant` | `H:300-303` | `Deleted` | removes `gr/export` and `es/current` |
| `ExportRecoveryKey`/`ExportProfile`/`ExportForSuccessor`/`DeleteIdentity`/`RepairIndex` | `consent::request` (`H:198-202`) | `RecoveryKey`/`Profile`/`ExportedSecrets`/`Deleted` or `Continue{remaining}`/`Ack{0}` | may answer later, after a prompt |
| inbox v2 ops | `inbox_v2::*` (`H:240-281`) | | other map |

Item ops pass `page_may` first (`H:310-320`, table `S:214-221`): `InboxV2`/`PassLedger` read-only,
`HeldPass` no access, `Inbound` read+delete.

**Identity record** (`H:337-354`): CBOR `{v:1, dsa, kem, vk, ek, alias, description, confirmed,
deleting}` in the generation envelope at `id/<addr hex>`.

**Create/import bookkeeping** (`H:580-667`): prune stale entries, write marker `dl/<addr>` = `c`,
claim the index slot (cap `MAX_IDENTITIES`), write the record, then delete the marker or set it to
`s`. Failures undo the claim or leave `f`. `retried` = three attempts (`H:1096-1098`); `partial`
turns any post-change failure into `Internal` (`H:1103-1112`).

**Deletion** (`H:922-1060`): marker `d` is written before the record is marked `deleting`
(`H:965-972`); each call removes at most `DELETE_STEP = 256` items (`H:913`) and answers
`Continue`; then tombstone shards, per-kind indexes and their markers, record, index entry, and
last the marker via `del_written` (`H:1032-1059`). An unreadable item index is reset and its items
abandoned (`H:1002-1011`).

**Exports** (`H:1604-1752`): recovery key text; profile = v0.2.x JSON backup `{version:2, alias,
description, keys:{ml_dsa_seed, ml_kem_seed}}`; successor export pages full keys and raw
enveloped values for `exportable_keys` (identity index minus deleting identities, records, item
indexes and items of `exported` kinds, non-empty tombstone shards), sorted by relative key,
cursor = last relative key (`H:1631-1676,1712-1760`).

### `S` (store.rs)
Caps (`S:36-56`): identities 16, contacts 2048, drafts 256, messages 4096 per kind per identity,
predecessors 256, origin budget 300,000 keys / 512 MiB, delete reserve 64 keys / 64 KiB, 4 pending
prompts, prompt TTL 600 s. Per-kind cap/shape tables at `S:184-240`.

Key layout, all under `o/<64 hex of origin>/` (`S:103-105`):

| Relative key | Holds | Enveloped |
|---|---|---|
| `id/<addr>` | identity record | yes |
| `ix/id`, `ix/<kind tag>/<owner or ->`, `ix/pg`, `ix/sn/<owner>/<nibble>` | index: CBOR list of ids | yes |
| `it/<kind tag>/<owner or ->/<id hex or ->` | item | yes |
| `xm/<index rel>` | index existence marker `p`/`l` (`S:77-87`) | no |
| `dl/<addr>` | identity marker `c`/`f`/`d`/`s` (`S:111-132`) | no |
| `pi/<16 bytes of blake3(item rel)>` | item pending marker (`S:309-312`) | no |
| `pg/<8 hex>` | stashed prompt (CBOR `Pending`) | no |
| `gr/export`, `es/current` | grant `b"1"`, export session | no |
| `mt/gen`, `mt/gen2`, `mt/pseq`, `mt/usage`, `mt/recount` | counters and accounting | no |
| `bg/cursor`, `bg/cursor-v2` | page job rotation (`D/src/watch.rs:53`, `D/src/inbox_jobs.rs:79`) | no |

Kind tags `S:145-163`. The all-zero origin holds `ix/bg/origins` and `bg/budget`
(`D/src/watch.rs:48-51`).

`Space` (`S:340-951`): `fetch` re-reads once on a miss (`S:365-367`). `put_within`
(`S:441-496`) reads the usage record; unknown means `recount()`; a growing write over budget
recounts before answering `CapReached` unless the finished recount already says the same
(`S:471-487`); usage is written before the value. Non-growing writes are never refused
(`S:458-465`). `del`, `del_written`, `remove_if_present` differ in what counts as proof of
removal (`S:504-560`). Recount: `known_keys` enumerates from the indexes (`S:613-671`),
`recount_step` reads `RECOUNT_STEP = 2048` keys per call and persists progress in `mt/recount`
(`S:682-708`); unreadable must-exist keys count 1024 bytes (`S:71`).
Envelope `[1][generation u64 LE][body]` (`S:279-294`). `next_generation` = max of the two
counter copies and the floor, plus one, written to both copies or it fails (`S:783-793`).
`write_if` is the CAS (`S:797-810`); `write_over_unusable` replaces in place only an unreadable or
undecodable value (`S:729-752`). Index protocol: `index` returns `Busy` when the record is absent
but its marker is `l` (`S:816-833`); creation is `p`, empty record, `l`, then the real list
(`S:889-920`); an emptied index stays as an empty list (`S:866-872`); `reset_index` only for an
unreadable one (`S:845-854`).

### `C` (consent.rs)
Three kinds (`C:77-95`): Export (choices `Allow`, `Always allow for this app`, `Deny`),
Destructive (`Delete`, `Cancel`), Repair (`Repair`, `Cancel`) (`C:43-48,505-509`).
`request` (`C:271-311`), in order: a deletion already in progress runs with no prompt; an
`ExportForSuccessor` carrying a session token goes to `continue_session`; `precheck` fails early
(missing identity, stale generation, readable index) (`C:227-269`); an Export with `gr/export`
present runs at once; otherwise `prune` then `raise_prompt`.
`raise_prompt` (`C:451-518`): bump `mt/pseq`; prompt id = first 4 bytes of a keyed BLAKE3 over
host random, request id, origin, seq, attempt (non-zero, not colliding); claim `ix/pg` then write
`pg/<id>` = CBOR `{seq, created, request_id, op}`; emit
`OutboundDelegateMsg::RequestUserInput{request_id: prompt id, message: JSON string, responses}`.
The delegate context slot is deliberately unused (`C:14-17`).
`answer` (`C:522-572`): look the stash up in the answering origin's namespace (a foreign origin
finds nothing and gets `[]`), delete it before acting, refuse if expired, then match the exact
choice bytes. `Always` writes `gr/export` first. Empty response (node timeout or dismissal) is
`Refused` "not answered" (`C:558-569`).
Expiry (`C:313-318`): `now - created > 600` or `created > now + 60`; with no clock, never.
`prune` (`C:400-449`) refuses expired stashes and evicts the oldest so at most 4 are live.
Export session (`C:121-215`): only when a clock exists; token = keyed BLAKE3 of host random,
request id, origin; stored as its hash with a 300 s sliding expiry, cursor, source generation and
10,000 pages (`C:51-53`).

### `K` (crypto.rs)
`Keys::from_seeds` (`K:28-35`): ML-DSA-65 `SigningKey::from_seed(32-byte seed)`, ML-KEM-768
`DecapsulationKey::new(64-byte seed)`; vk 1952 bytes, ek 1184 bytes. `sign` uses the empty context
(v1), `sign_v2` a context, deterministic (`K:48-62`). `MixRng` = BLAKE3 XOF keyed
`"freenet-mail delegate rng v1"` over purpose, host bytes, caller entropy (`K:80-97`);
`new_seeds` takes 32 then 64 bytes (`K:116-118`), purpose `"create-identity"` (`H:683`).
Address = first 24 bytes of `blake3::derive_key("freenet-mail address v1", vk)`
(`protocol/src/address.rs:4-12`). Recovery key = `fmail-key-1:` + base58 of
`[0x01][dsa 32][kem 64][sha256(prefix||body)[..4]]` (`protocol/src/recovery_key.rs:6-17`).
v1 message: `nonce 24 | ML-KEM ct 1088 | XChaCha20-Poly1305`, key = HKDF-SHA256(shared, info
`"freenet-email-v1 message-key"`) (`K:16-19,120-177`).

### `E` (env.rs)
`Env` (`E:15-30`): `get`, `set`, `remove`, `host_random`, `now_secs`, `contract_state`, `log`.
`DelegateCtx` maps them to `get_secret`, `set_secret`, `remove_secret`,
`freenet_stdlib::rand::rand_bytes(32)`, `freenet_stdlib::time::now()` (wasm only; `None`
natively), `get_contract_state`, `freenet_stdlib::log::info` (`E:32-66`). No listing, no
`has_secret`, no context slot, no `create_delegate`.
`mem::MemEnv` (`E:262-324`) is the test host: a `BTreeMap`, counter-derived randomness, settable
clock and contracts, `fail_writes`, `fail_keys_containing`, per-key write counts, and `Faults`.

## 3. Interfaces

- **Wire:** CBOR (`ciborium`) `Request{version:1, request_id:[16], op}` in
  `ApplicationMessage.payload`; one `Response{request_id, result: Result<Reply, Failure>}` back
  (`P:104-121`). `FailureCode` at `P:123-144`. Encodings pinned per variant in
  `protocol/src/golden_v1.txt` (append-only, `protocol/src/golden.rs:1-13`).
- **Other crates used:** `freenet-mail-wire-v1` (`inbox::{InboxSettings, sign_payload,
  InboxParams, STATE_UPDATE, removal_payload}`, `aft::DelegateParameters`); the file
  `published-contract/wasm/v1/inbox_code_hash` is compiled in (`H:1466-1468`).
- **Callers:** node FFI; UI offline build (`ui/Cargo.toml:41,127`, always with `v1-transition`);
  UI node build embeds `published-contract/candidate/mail_delegate`
  (`ui/src/mail_delegate/node.rs:19-22`) and sends `DelegateRequest::ApplicationMessages`
  (`:225-233`). In-crate, `inbox_v2.rs`/`watch.rs` call `handlers::put_item` directly, bypassing
  `page_may`.
- **UI couplings:** `DELEGATE_PROMPT_TTL_SECS = 600` pinned to `PROMPT_TTL_SECS`
  (`ui/src/mail_delegate/mod.rs:75,347-354`); the offline build answers every prompt with its
  **first** choice (`ui/src/mail_delegate/local.rs:144-153`), so the order at `C:505-509` matters.
- **cfg:** `feature = "v1-transition"` (`H:468,1584-1600`), `target_family = "wasm"` (`E:51-58`),
  `#![allow(unexpected_cfgs)]` for the macro (`L:17`).

## 4. Invariants and where they are held

| Rule | Code | Pinned by |
|---|---|---|
| Non-web callers change nothing | `H:123-147` | `T:227-252` |
| One origin cannot read or change another | `S:103-105,352` | `T:258-336`, `S:959` |
| Absence believed only after a re-read | `S:365-367` | fault sweep `TF:1040` |
| Generations never reused; both counter copies written | `S:783-793` | `S:989,1022`, `T:450,1702`, `TF:948-954` |
| CAS on every enveloped write | `S:797-810`, `H:1181-1196` | `T:424,486`, `S:1360` |
| Unreadable index is `Busy`, never rebuilt | `S:816-833` | `S:1041,1268,1306` |
| Only marker `d` licenses finishing a delete unprompted; `c`/`s` license nothing | `H:984-993,498-520` | `TF:1129`, `TR:593`, `TS:194` |
| Approval recorded before the record is touched; restore cannot undo it | `H:965-972`, `H:771-774` | `TR:535`, `T3:193` |
| Stash claimed before written, deleted before answered: at most one response | `C:496-503,530-540` | `T2:270-349`, `T3:243-301`, `TF:204-212` |
| Prompts and sessions bound to their origin | `C:526-529,176-189` | `T:1324-1330`, `T3:87` |
| Export grant covers exports only | `C:302` | `T:1505-1543` |
| Budget refuses growth only; deletes use the reserve | `S:458-465,437` | `S:1215`, `TR:127` |
| Lost or drifted usage is recounted | `S:466-487` | `TR:712,733,753`, `TF:962` |
| Successor export has no bookkeeping, no migration records, no deleting identities | `H:1631-1702`, `S:206` | `T:1574,1630`, `T3:211` |
| v1 keys, signatures, envelope byte-compatible | `K:28-62,149-177` | `K:209-226`, `T:961-1095` |
| Removal signer cannot be made to sign a v1 message or v2 layout | `H:1526-1580` | `T:1134` |

## 5. State

- **Candidate, not frozen.** `published-contract/candidate/mail_delegate` (2.7 MB, code hash
  `Bavf78hk…`), built with `freenet-main-delegate,v1-transition`
  (`published-contract/candidate/README.md`). `published-contract/wasm/v2/` does not exist.
  `.release-freeze` blocks releases until the v0.3.0 chain is complete.
- **Not releasable as is:** `scripts/check-release-delegate.sh` requires a frozen v2 build without
  `v1-transition` and no UI reference to `candidate/`; the UI still calls `V1AftParams`
  (`ui/src/aft.rs:483`).
- **Frozen inputs it must keep matching:** v1 crypto vectors, `fmail-key-1:` format, v0.2.x JSON
  backup, `protocol/src/golden_v1.txt`.

## 6. Tests

All native, driven through `handlers::process` over `MemEnv` (`T:45-79`). 115 tests in the owned
files: `T` 35, `TF` 8, `TR` 21, `TS` 8, `T2` 11, `T3` 9, inline `S` 17, inline `K` 6.

- `T`: gate, namespace isolation, caps, CAS, listing/paging, identity create/import/rename, v1
  crypto against v0.2.x vectors (`ui/src/inbox/crypto_vectors_v1.txt`,
  `ui/src/app/testdata/identity-backup-v2.json`), removal signer, consent basics, successor
  export, version refusal.
- `TR`: recovery from one persistently unreadable key (identity record, item, identity index,
  item index), corrupt records, markers, usage recount.
- `TS`: interrupted item/identity deletes, typed answers to a newer UI, pruning rules.
- `T2`/`T3`: export sessions (binding, sliding expiry, clock-skew bound, page limit, revoke),
  paged deletion, one response under failing prompt bookkeeping.
- `TF`: seeded random runs. `Run::step` picks among create, rename, delete identity, import, put,
  watch step, delete item, list (`TF:260-272`) against a host where each get/set/remove fails with
  probability `FAULT_RATE` per mille and replies are lost at the same rate; a transient miss is
  hidden by the immediate re-read (`E:248-256`); the "blind" variant makes one present key
  permanently unreadable (`E:242-246`). `heal` turns faults off, wakes `rescan` four times,
  finishes deletions and restores unlisted identities (`TF:664-749`); `check` (`TF:786-956`) then
  requires: every stored identity listed and every entry backed by a record, no leftover `dl/`
  marker beside an entry, started deletions never undone, deleted identities leave no key naming
  them (except items under a blinded list), every item indexed and every index entry stored, each
  stored value is one some reply allowed, confirmed-deleted kept copies never return, no
  generation handed out twice, and a write succeeds after a forced-full usage record.
  Defaults: 300 seeds x 60 ops at 60 per mille; blind rate 20 (`TF:1041-1043,1086`); pinned seeds
  `TF:28-53`.

Commands (toolchain 1.99.1 from `rust-toolchain.toml`; needs the committed candidate WASM, which
`D/src/tests/jobs.rs:173-178` reads):
```
cd delegates/mail-delegate && cargo test --locked
cd delegates/mail-delegate && cargo test --locked --features v1-transition
FAULT_SEEDS=2000 FAULT_OPS=80 FAULT_RATE=200 FAULT_BLIND_RATE=20 cargo test --release --locked --lib \
  --features v1-transition tests::faults::invariants_hold_while_a_key_stays_unreadable -- --exact
FAULT_SEED=n FAULT_TRACE=1 cargo test replay_one_unreadable_key_seed     # one seed
```
`cargo make test` reaches both feature sets through `scripts/check-excluded-crates.sh test`
(`Makefile.toml:907-924`); CI runs the 20% sweep on every PR (`.github/workflows/build.yml:113-129`).

## 7. Changing it safely

1. **Any source edit changes the WASM, hence the delegate key.** Rebuild the candidate and its
   code hash with `scripts/build-wasm.sh` on linux/amd64 or take CI's artifact
   (`published-contract/candidate/README.md`); `scripts/check-delegate-candidate.sh` is in the
   required `check-drift`. Test-only edits can change the bytes too. Nothing migrates data between
   two candidate builds.
2. **Guards are textual.** `scripts/check-guards-fail.sh` applies exact-string mutations to these
   files: 69 guards anchor here (37 in `H`, 20 in `S`, 11 in `C`, 1 in `E`; none in `K` or `L`),
   e.g. `:323-333`, and `:1730-1732` anchors on `MAX_PENDING_PROMPTS`. Rewording an anchored line
   breaks the guard; run `--anchors`. A guard on a new directory needs an `area_of` entry in
   `scripts/select-guards.sh`. `crypto.rs` and the manifest have no mutation guard: only the
   vector tests and `D/src/tests/jobs.rs:172-200` protect them.
3. **Protocol changes** go in `protocol/`: add the variant to the exhaustive `*_name` matches and
   append a golden line; never edit one. Handle it in `H:196-304`, in `C:86-116` if consented, and
   in the UI's `timeout_ms`/`consented` lists (`ui/src/mail_delegate/mod.rs:81-112`).
4. **New item kind:** extend `kind_tag`, `ALL_KINDS` (array length), `shape`, `cap`, `exported`,
   `page_access` (`S:145-240`) and `prompt_text` (`C:356-370`); `delete_step`, `known_keys` and
   `exportable_keys` follow `ALL_KINDS`. Check the origin key budget (section 9).
5. **New stored key outside the index scheme** must be added to `known_keys` (`S:613-671`), and
   considered for `exportable_keys` and `delete_step`.
6. **Storage layout is persistent data** once frozen: envelope tag, key formats, marker letters,
   `IdentityRecord` fields (`#[serde(default)]` for additions, as `deleting`), CBOR index lists.
7. **Never conclude absence from one read**, never unlist on a missing record without a marker,
   write markers before the thing they license. Run both sweeps at a high rate after touching
   `S` or any marker logic; add failing seeds to `PINNED_*`.
8. **Prompt choice order and strings** are matched byte-for-byte (`C:549-557`) and the offline
   build takes the first.
9. **Host imports:** a new stdlib call must be in `scripts/delegate-imports-allowlist.txt`
   (pinned to stdlib 0.12.1's set) or the node will not instantiate the module.
10. **Wake-ups are at the stdlib maximum of 4**; tags are pinned by
    `D/src/tests/jobs.rs:172-200` against the committed WASM.

## 8. Disagreements between docs/comments and code

1. AGENTS.md: keys leave only by approved export, "with one exception" (`CreateIdentity`).
   Code: with `v1-transition`, which the committed candidate and the offline build both carry,
   `V1AftParams` returns the ML-DSA seed with no prompt (`H:1587-1595`, `ui/Cargo.toml:41,127`).
2. README.md:40-42,77-80 says keys live in the "identity delegate"
   (`modules/identity-management/`), "never exposed to the webapp". The UI registers the mail
   delegate (`ui/src/mail_delegate/node.rs:19-22`) and receives the recovery key on creation.
3. `D/Cargo.toml:62-64`: `v1-transition` "keeps the operations the v0.2.x send path needs".
   It gates one operation; `DecryptV1`, `SealV1Message`, `SignV1Authority`, `SignV1Removal` are
   unconditional (`H:282-297`).
4. `S:43-45`: the 300,000-key budget "fits the per-kind caps" for 16 identities. Current caps
   sum to about 436,000 items (26,882 per identity x 16 + 6,402 unowned) before indexes, markers
   and tombstones.
5. AGENTS.md: "every record ... carries a generation". Markers, `mt/*`, `gr/export`,
   `es/current`, `pg/*`, `bg/*` are raw values (`C:155-158,500,552`).
6. `H:89-91` error text says only application messages and prompt answers are accepted; six host
   message kinds are handled just above.
7. `S:16-28` key table omits `dl/`, `pi/`, `xm/`, `mt/gen2`, `mt/recount`, `ix/sn/`, `bg/`.
8. `S:1059` test comment "Emptied and removed" contradicts `S:866-872`.
9. `P:618-621`: creation is "the one time" a recovery key leaves without a prompt; the Always
   grant also skips it (`C:302-304`).
10. AGENTS.md "every request gets exactly one response": not for an undecodable envelope, a
    `processed` message (`H:42-50`) or a corrupt stash (`C:541-542`), which return `DelegateError`.
11. The doc comment describing the removal signer is attached to `const V2_DOMAIN`
    (`H:1501-1524`), not to `sign_v1_removal`.

## 9. Suspected bugs and weak spots

1. **Unreadable `mt/usage` on a store with more than 2048 known keys blocks all writes
   (medium).** Each `put` that finds usage unknown calls `recount_step`; after a finished recount
   the state is `DONE`, so the next `put` in the same call restarts from zero and returns `Busy`
   (`S:466-469,684-708`). Every write needs at least three puts (`S:790-791,808`). Only bites if
   the key stays unreadable after being rewritten; the sweeps use stores far below 2048 keys.
2. **`known_keys` omits `bg/cursor` and `bg/cursor-v2` (high, tiny impact).** They are written
   through `Space::put` (`D/src/watch.rs:115`, `D/src/inbox_jobs.rs:269`), so a recount
   under-counts, against `S:575-578`.
3. **Origin key budget below the sum of kind caps (high on arithmetic).** See 8.4.
4. **Unreadable `ix/pg` cannot be repaired (high on reading, low likelihood).** Every prompt
   needs it (`C:402,499`), `RepairIndex` needs a prompt and can only name `ix/id` or an item-kind
   index (`H:888-902`).
5. **Expired prompts are refused only when the same origin raises another prompt or an answer
   arrives** (`prune` has one call site, `C:305`). If the node never sends a `UserResponse`, the
   request is never answered and the stash stays (medium; depends on core).
6. **A recount step is not bounded by `RECOUNT_STEP`:** `known_keys()` runs in full on every step
   (`S:683`), reading every index and hashing every listed item (low-medium).
7. **Identity deletion never removes `pi/` markers** of the identity's items
   (`H:1014-1023`); leftovers exist only after failed bookkeeping (low).
8. **Behaviour to know:** `DeleteItem` of a missing Kept/Archived/Flags id still writes a
   tombstone and can evict an older one (`H:1255-1265`, `D/src/watch.rs:79-84`); a `UserRestore`
   import of an already-present unconfirmed identity leaves it unconfirmed (`H:756-768`);
   `Status.identities` counts deleting and unreadable entries.

## 10. Open work and documented limits

- Remove `v1-transition` and the v1 ops with the v1 send path (`D/Cargo.toml:62-64`); freeze into
  `published-contract/wasm/v2/`; lift `.release-freeze`.
- No importer for a mail-delegate successor exists; `H:1704-1711` states what one must do with
  generations.
- Limits stated in code: a key that is unreadable and whose removal is always refused looks
  absent (`S:507-512`); items under an unreadable list survive identity deletion as unreachable
  bytes (`H:1002-1011`); usage bytes run high (`S:570-581`); without a host clock prompts never
  expire and exports have no session (`C:119-120,313-318`); tombstone shards cannot be repaired
  (AGENTS.md).
- `Reply::Identity` is on the wire but never produced; `prune`'s `make_room` is always `true`.
- No TODO/FIXME markers in the owned files.

## 11. Questions not answerable from the tree

1. Does freenet-core deliver `UserResponse` with the prompting app's `WebApp` origin, and always
   send an empty one on timeout? The gate (`H:54`) and issue 9.5 depend on it.
2. Does rewriting a secret the node could not read make it readable again (issue 9.1)?
3. Can a client inject `WakeupFired`? stdlib documents that only for `Lifecycle`.
4. Is it intended that each candidate rebuild orphans data stored under the previous candidate?
5. The decision records and review rounds cited in comments ("decision 24", "B1", "Y1-Y5",
   "Round 10") are not in the repository.

---

## Appendix: the reader's structured summary

The mail delegate (delegates/mail-delegate, v0.3.0, outside the workspace with its own lockfile) is one WASM delegate holding every identity's ML-DSA-65/ML-KEM-768 seeds, contacts and local mail state. The core mapped here is: the single entry point and message classification (handlers::process), the web-app origin gate, a CBOR request/response dispatch over about 29 operations, identity create/import/rename/delete, generic item storage with per-kind caps, v0.2.x (v1) crypto, consented exports and deletion (consent.rs), a per-origin namespaced store with budgets, a usage record with paged recount, compare-and-swap generations and self-maintained indexes (store.rs), and a thin host trait (env.rs) with an in-memory fault-injecting test host.

The code is defensive around one host weakness: a failed read looks like an absent key. Markers, double reads and "write over, never remove first" rules exist for that, and seeded fault sweeps check the resulting invariants. 115 tests live in the owned files; 69 mutation guards in scripts/check-guards-fail.sh anchor on exact source text in handlers.rs, store.rs, consent.rs and env.rs.

State: a candidate, not frozen. The committed WASM is built with the v1-transition feature, which exposes the ML-DSA seed through V1AftParams without a prompt; a release must drop that feature, freeze the build into published-contract/wasm/v2 and remove .release-freeze. I found no crash-level bug, but several weak spots: a possible permanent Busy when the usage record stays unreadable on a large store, an unrepairable prompt index, a key budget smaller than the sum of kind caps, and small accounting leaks. AGENTS.md and README.md overstate the "keys never leave" rule.

### Key facts

- Entry point: lib.rs forwards to handlers::process and ignores parameters (delegates/mail-delegate/src/lib.rs:47-54); the delegate key is blake3(blake3(wasm) || []) because the UI registers it with empty parameters (ui/src/mail_delegate/node.rs:28-32).
- Manifest: lifecycle=[NodeStarted], capabilities=[Background], wakeups rescan=900, v1watch=3600, renew=21600, clock=86400 (delegates/mail-delegate/src/lib.rs:41-45); tags must equal jobs::Tick::ALL (delegates/mail-delegate/src/jobs.rs:82-87) and stdlib 0.12.1 allows at most 4 wake-ups, so the manifest is full. A test reads them from the committed WASM (delegates/mail-delegate/src/tests/jobs.rs:172-200).
- Message classification (delegates/mail-delegate/src/handlers.rs:34-93): ApplicationMessage and UserResponse pass check_origin; NodeStarted, WakeupFired, ContractNotification and GetContractResponse start a background run with no origin check and their payload is never read; SubscribeContractResponse and UpdateContractResponse are ignored; everything else (PutContractResponse, DelegateMessage, UnsubscribeContractResponse, Lifecycle::Installed) returns DelegateError.
- Origin gate (delegates/mail-delegate/src/handlers.rs:123-147): only Some(MessageOrigin::WebApp(32-byte id)) that is not all-zero (watch::GLOBAL, delegates/mail-delegate/src/watch.rs:48) is served. Delegate(_), None and unknown variants get Err(DelegateError::Other) with no protocol response and no storage access.
- Wire: CBOR Request{version:1, request_id:[16], op} decoded first as RequestEnvelope so a wrong version gets UnsupportedVersion and an undecodable op (including a malformed known op) gets UnknownOperation, both with the request id (delegates/mail-delegate/src/handlers.rs:167-195; protocol/src/lib.rs:656-671). Encodings are pinned append-only in protocol/src/golden_v1.txt (protocol/src/golden.rs:1-13).
- Dispatch table is one match at delegates/mail-delegate/src/handlers.rs:196-304. Five ops go to consent::request and may answer later (ExportRecoveryKey, ExportProfile, ExportForSuccessor, DeleteIdentity, RepairIndex, :198-202); item ops are filtered by page_may using store::page_access (delegates/mail-delegate/src/store.rs:214-221): InboxV2/PassLedger read-only, HeldPass no access, Inbound read+delete.
- Namespace: every key is o/<64 hex origin>/<relative key> (delegates/mail-delegate/src/store.rs:103-105). Relative keys: id/<addr>, ix/id, ix/<kind tag>/<owner or ->, it/<kind tag>/<owner or ->/<id>, ix/pg, pg/<8 hex>, ix/sn/<owner>/<nibble>, xm/<index>, dl/<addr>, pi/<hash>, gr/export, es/current, mt/gen, mt/gen2, mt/pseq, mt/usage, mt/recount, plus bg/cursor and bg/cursor-v2 written by the jobs (store.rs:60-142,246-312; watch.rs:53; inbox_jobs.rs:79).
- Caps and budgets (delegates/mail-delegate/src/store.rs:36-56): 16 identities, 2048 contacts, 256 drafts, 4096 per message kind per identity, 300,000 keys and 512 MiB per origin, 64 keys / 64 KiB reserve for delete bookkeeping, 4 pending prompts, prompt TTL 600 s. Item body max 256 KiB, response max 512 KiB (protocol/src/lib.rs:28-34).
- Usage accounting: put_within reads mt/usage; unknown triggers a recount; a growing write over budget recounts before CapReached; non-growing writes are never refused (delegates/mail-delegate/src/store.rs:441-496). Recount enumerates known_keys from the indexes and reads RECOUNT_STEP=2048 keys per call, answering Busy until done (store.rs:613-708).
- Compare-and-swap: enveloped records are [1][generation u64 LE][body] (delegates/mail-delegate/src/store.rs:279-294); next_generation takes max(mt/gen, mt/gen2, floor)+1 and must write both copies (store.rs:783-793); write_if is the CAS (store.rs:797-810). PutItem: None = create-only, Some(g) = CAS, an identical-body retry is acked (handlers.rs:1181-1196).
- Indexes replace host listing: index() returns Busy when the record is absent but marker xm/<rel> is 'l' (delegates/mail-delegate/src/store.rs:816-833); an emptied index is kept as an empty list (store.rs:866-872); only reset_index (after a RepairIndex prompt) or identity deletion removes one (store.rs:845-854).
- Markers c/f/d/s (delegates/mail-delegate/src/store.rs:111-132) are the only licence to drop an index entry whose record reads absent; only 'd' lets a deletion continue without a prompt (handlers.rs:984-993). Deletion writes 'd' before marking the record deleting and proceeds 256 items per call, answering Continue (handlers.rs:913,922-1060).
- Consent flow (delegates/mail-delegate/src/consent.rs:271-311,451-572): the request is stashed as CBOR at pg/<prompt id> in the caller's namespace after claiming ix/pg, the host gets OutboundDelegateMsg::RequestUserInput with choices [Allow, Always allow for this app, Deny] / [Delete, Cancel] / [Repair, Cancel]; the answer arrives as UserResponse under the same origin, the stash is deleted before acting, and an empty answer is Refused. 'Always' writes gr/export, which skips prompts for all exports of that origin until RevokeExportGrant.
- Prompt expiry needs a host clock: expired if now-created > 600 s or created > now+60 s; with no clock never (delegates/mail-delegate/src/consent.rs:313-318). Export sessions (300 s sliding, 10,000 pages, token hash stored) also need the clock (consent.rs:51-53,121-215). The UI pins its 600 s constant to the delegate's (ui/src/mail_delegate/mod.rs:75,347-354).
- Key derivation: seeds = BLAKE3-XOF MixRng('create-identity', host_random, ui_entropy), 32 bytes ML-DSA seed then 64 bytes ML-KEM seed (delegates/mail-delegate/src/crypto.rs:80-118; handlers.rs:682-684); address = first 24 bytes of blake3::derive_key('freenet-mail address v1', vk) (protocol/src/address.rs:4-12); recovery key = 'fmail-key-1:' + base58([0x01][dsa 32][kem 64][sha256 checksum 4]) (protocol/src/recovery_key.rs:6-17). CreateIdentity returns it once (handlers.rs:692-697).
- v1-transition gates exactly one operation, V1AftParams, which returns JSON containing the ML-DSA seed with no prompt, plus a marker string and Status.v1_transition (delegates/mail-delegate/src/handlers.rs:468,1584-1600). The committed candidate and the UI's offline build both enable it (published-contract/candidate/README.md; ui/Cargo.toml:41,127) and the UI calls it (ui/src/aft.rs:483).
- Env wraps exactly: get_secret, set_secret, remove_secret, rand::rand_bytes(32), time::now() (wasm only, None natively), get_contract_state, log::info (delegates/mail-delegate/src/env.rs:15-66). No listing, no context slot. New host imports must be in scripts/delegate-imports-allowlist.txt.
- Fault host: env::mem::Faults injects per-mille get misses (hidden by the immediate re-read), set/remove refusals, targeted nth-call rules, and one persistently unreadable key (delegates/mail-delegate/src/env.rs:75-257). tests/faults.rs runs seeded op mixes, heals, then checks listing/record agreement, no leaked slots, no lost acknowledged write, no reused generation, never stuck at CapReached (faults.rs:786-977). Defaults 300 seeds x 60 ops at 6%; CI adds 2000 seeds at 20% (.github/workflows/build.yml:113-129).
- Changing any source file changes the WASM and so the delegate key: rebuild published-contract/candidate/mail_delegate and its code hash with scripts/build-wasm.sh (scripts/check-delegate-candidate.sh is in required check-drift). 69 guards in scripts/check-guards-fail.sh mutate exact text in handlers.rs (37), store.rs (20), consent.rs (11), env.rs (1); none cover crypto.rs or lib.rs.
- The offline UI build answers every prompt with its first choice (ui/src/mail_delegate/local.rs:144-153), so the order of responses at delegates/mail-delegate/src/consent.rs:505-509 is load-bearing, and choice strings are matched byte-for-byte (consent.rs:549-557).

### Docs or comments that disagree with the code

- **[medium]** Claim (AGENTS.md, section 'The mail delegate', first paragraph): Keys leave the delegate only through an export the user approves, with one exception: CreateIdentity returns the recovery key once. **Reality:** With the v1-transition feature, which the committed candidate and the UI's offline build both enable, V1AftParams returns the v1 token-generator parameters containing the identity's ML-DSA seed to the page with no prompt. The 'Always allow' grant also releases recovery keys without a prompt. Evidence: delegates/mail-delegate/src/handlers.rs:1587-1595
- **[medium]** Claim (README.md:40-42 and README.md:77-80): Private keys live inside the 'identity delegate' (modules/identity-management/) and are never exposed to the webapp. **Reality:** The UI embeds and registers the mail delegate candidate (delegates/mail-delegate); AGENTS.md itself says the UI no longer runs identity-management. The recovery key is returned to the page on creation. Evidence: ui/src/mail_delegate/node.rs:19-22
- **[low]** Claim (delegates/mail-delegate/Cargo.toml:62-64): v1-transition 'keeps the operations the v0.2.x (v1 inbox) send path needs'. **Reality:** Only V1AftParams (and a marker constant and a status flag) is gated. DecryptV1, SealV1Message, SignV1Authority and SignV1Removal are compiled unconditionally. Evidence: delegates/mail-delegate/src/handlers.rs:282-298
- **[medium]** Claim (code comment, delegates/mail-delegate/src/store.rs:43-45): MAX_ORIGIN_KEYS (300,000) 'fits the per-kind caps for MAX_IDENTITIES identities (16 x (256 + 4 x 4096 + 1) + 2048 + ...)'. **Reality:** With the kinds added since (PassLedger 4096, Inbound 4096, HeldPass 2048, InboxV2 1 per identity; LegacyOrphan 4096, MigrationMarker 256) the caps sum to about 436,000 items (26,882 per identity x 16 + 6,402 unowned) before index records, markers and tombstones. Evidence: delegates/mail-delegate/src/store.rs:224-240
- **[low]** Claim (AGENTS.md, 'The mail delegate', Compare-and-swap bullet): Compare-and-swap: every record, index records included, carries a generation that is checked on write. **Reality:** Only identity records, items and indexes are enveloped. Markers (dl/, pi/, xm/), mt/*, gr/export, es/current, pg/* stashes and bg/cursor are raw values written with Space::put and no generation. Evidence: delegates/mail-delegate/src/consent.rs:155-158
- **[low]** Claim (AGENTS.md, 'The mail delegate', Responses bullet; delegates/mail-delegate/src/consent.rs:17-21): Every request gets exactly one response, including a typed one for an unknown version or operation. **Reality:** A request whose envelope does not decode, one flagged processed, or one from a non-web origin returns DelegateError and no Response; a corrupt stash is deleted and then returns DelegateError. An expired prompt is refused only when the same origin raises another prompt or an answer arrives. Evidence: delegates/mail-delegate/src/handlers.rs:41-50
- **[low]** Claim (error text, delegates/mail-delegate/src/handlers.rs:89-91): 'the mail delegate accepts only application messages and prompt answers' **Reality:** The same match handles Lifecycle(NodeStarted), WakeupFired, ContractNotification, GetContractResponse, SubscribeContractResponse and UpdateContractResponse just above. Evidence: delegates/mail-delegate/src/handlers.rs:62-88
- **[low]** Claim (module doc, delegates/mail-delegate/src/store.rs:16-28): The module's table of relative keys. **Reality:** It omits dl/<addr>, pi/<hash>, xm/<index>, mt/gen2, mt/recount, ix/sn/<owner>/<nibble> and the bg/ keys, all of which the code writes. Evidence: delegates/mail-delegate/src/store.rs:60-142
- **[low]** Claim (test comment, delegates/mail-delegate/src/store.rs:1059): 'Emptied and removed, it is absent again, not unreadable.' **Reality:** An emptied index keeps its record as the empty list under marker 'l'; it is not removed. Evidence: delegates/mail-delegate/src/store.rs:866-872
- **[low]** Claim (protocol/src/lib.rs:618-621): CreatedIdentity is 'the one time a recovery key leaves the delegate without a prompt'. **Reality:** With the remembered 'Always allow for this app' grant, ExportRecoveryKey and ExportProfile answer with the key and no prompt. Evidence: delegates/mail-delegate/src/consent.rs:302-304
- **[low]** Claim (AGENTS.md, 'The mail delegate', Consent bullet): Deleting an identity 'removes everything it owns'. **Reality:** Items under an index that cannot be read are abandoned as unreachable bytes (stated in a code comment), and pi/ pending markers of the identity's items are never removed by delete_step. Evidence: delegates/mail-delegate/src/handlers.rs:1002-1023
- **[low]** Claim (delegates/mail-delegate/src/handlers.rs:1501-1523): Doc comment 'Sign a v1 RemoveMessages for messages in the identity's own inbox ...' **Reality:** The comment is attached to the constant V2_DOMAIN, not to sign_v1_removal, so rustdoc shows the signer's contract on the wrong item and the function has none. Evidence: delegates/mail-delegate/src/handlers.rs:1524-1526

### Suspected issues (leads from reading, not confirmed)

- **A persistently unreadable mt/usage record blocks every write on a store with more than 2048 known keys** (confidence: medium). put_within calls recount() whenever the usage record reads as absent. recount_step, finding the recount state DONE, restarts from zero; with more than RECOUNT_STEP known keys it returns Busy. If mt/usage stays unreadable after set_usage rewrites it, the first put of a call can finish a recount but the second put in the same call restarts it and fails. Every write needs at least three puts (two counter copies and the value), so no write completes. Read from the code; not exercised, because the fault sweeps use stores far smaller than 2048 keys. It only matters if a rewritten key stays unreadable on a real node, which I could not check. Evidence: delegates/mail-delegate/src/store.rs:466-469
- **known_keys omits bg/cursor and bg/cursor-v2, so a recount under-counts** (confidence: high). Both keys are written through Space::put into the page's own namespace and are counted by the usage record, but known_keys does not list them, so after any recount the record is low by up to two keys. Tiny impact, but it contradicts the stated rule that the record never runs low. The delegate's own namespace (bg/budget, ix/bg/origins) is uncounted the same way. Evidence: delegates/mail-delegate/src/store.rs:613-623
- **Origin key budget is smaller than the sum of per-kind caps** (confidence: high). 300,000 keys per origin against roughly 436,000 items allowed by the per-kind caps for 16 identities, before index records, markers and tombstone shards. A heavy user hits 'this app's storage budget is full' (after a multi-call recount answering Busy) rather than a per-kind cap. The arithmetic is certain; whether it is intended is not. Evidence: delegates/mail-delegate/src/store.rs:43-45
- **An unreadable prompt index (ix/pg) cannot be repaired and blocks every operation that must prompt** (confidence: medium). prune returns nothing and raise_prompt's index_add fails with Busy when ix/pg is unreadable. RepairIndex itself needs a prompt, and repair_target can name only the identity index or an item-kind index. New deletions, repairs and exports without an 'Always' grant would answer Busy for good for that origin. Evidence: delegates/mail-delegate/src/handlers.rs:888-902
- **Expired prompts are only refused lazily** (confidence: medium). prune has a single call site, inside consent::request when a new prompt is about to be raised. If the node never delivers a UserResponse for a prompt, the waiting request gets no Response from the delegate until the same origin asks for another consented operation; the stash, which holds the full Op, stays stored. The design relies on the node sending an empty answer at its own timeout. Evidence: delegates/mail-delegate/src/consent.rs:305
- **A recount step is not bounded by RECOUNT_STEP** (confidence: low). recount_step calls known_keys() in full on every call. That reads every index of every identity and computes a blake3 pending-marker key for every listed item before the 2048 bounded value reads begin, so the per-call cost grows with the whole store, against the comment that one call stays well within the host's time limit. Evidence: delegates/mail-delegate/src/store.rs:683
- **Identity deletion leaves pi/ pending markers behind** (confidence: medium). delete_step removes each item and its index entry but never the item's pending marker. A marker exists only after a failed create or delete (c, f, s or d left behind), so the leak is small; the leftover keys name no identity and are uncounted after the next recount. Evidence: delegates/mail-delegate/src/handlers.rs:1014-1023
- **DeleteItem on a missing Kept/Archived/Flags id still writes a tombstone** (confidence: medium). When the record reads as absent the delete skips the generation check, calls keep_out_of_the_watch and answers Deleted. A page can therefore tombstone any 16-byte id for its own identities, and a full shard (512) evicts an older tombstone. Same-origin only, so not a privilege issue, but it weakens 'a deleted message stays gone' under a buggy or hostile page script. Evidence: delegates/mail-delegate/src/handlers.rs:1255-1265
- **A malformed known operation is reported as UnknownOperation** (confidence: high). Any failure to deserialize the op value maps to UnknownOperation, so the UI cannot tell a bad argument from a delegate that is too old for the operation. Evidence: delegates/mail-delegate/src/handlers.rs:184-195
- **Status.identities counts deleting and unreadable identities** (confidence: high). status reports the length of the identity index, while ListIdentities leaves out identities being deleted and records that cannot be used; the two numbers can disagree. A UserRestore import of an identity already stored unconfirmed also returns it unchanged, still unconfirmed. Evidence: delegates/mail-delegate/src/handlers.rs:463

### Open work

- Remove the v1-transition feature and V1AftParams with the v1 send path ('Removed with that path when the v2 inbox lands'): delegates/mail-delegate/Cargo.toml:62-64; the UI still depends on it at ui/src/aft.rs:483.
- Freeze the candidate into published-contract/wasm/v2/ (directory does not exist) and point the UI at it; scripts/check-release-delegate.sh refuses a release of 0.3.0 or later until then (scripts/check-release-delegate.sh:68-85).
- Releases are frozen by .release-freeze until the v0.3.0 chain is complete (.release-freeze:1-11).
- No importer exists for a mail-delegate successor; export_for_successor documents that one must start its generation counter above the largest imported generation (delegates/mail-delegate/src/handlers.rs:1704-1711).
- Documented limit: a key that cannot be read and whose removal is refused every time is indistinguishable from an absent one (delegates/mail-delegate/src/store.rs:507-512).
- Documented limit: items named by an unreadable index survive identity deletion as unreachable bytes (delegates/mail-delegate/src/handlers.rs:1002-1011).
- Documented limit: the usage record's bytes run high when an unreadable key is removed or replaced (delegates/mail-delegate/src/store.rs:570-581).
- Documented limit: without a host clock prompts never expire and a paged export asks on every page (delegates/mail-delegate/src/consent.rs:119-120 and 313-318).
- Reply::Identity is defined and pinned on the wire but no handler produces it (protocol/src/lib.rs:578); consent::prune's make_room parameter is always true (delegates/mail-delegate/src/consent.rs:305,400).
- No TODO or FIXME markers in any owned file.

### Open questions

- Does freenet-core deliver InboundDelegateMsg::UserResponse with the prompting app's MessageOrigin::WebApp, and does it always send an empty response when a prompt times out? The gate at delegates/mail-delegate/src/handlers.rs:54 and the 'exactly one response' rule both depend on it; the tree cannot show it.
- On a real node, does rewriting a secret the node could not read or decrypt make it readable again? That decides whether the mt/usage wedge (suspected issue 1) can happen outside the test host's model.
- Can a client inject WakeupFired through ApplicationMessages? stdlib documents that the host refuses Lifecycle from clients but says nothing equivalent for WakeupFired; process() applies no origin check to either.
- Is it intended that every rebuild of the candidate (a new delegate key) orphans data stored under the previous candidate build during development? Nothing in the tree migrates between two mail-delegate builds.
- Is the 300,000-key origin budget meant to bind before the per-kind caps now that inbox v2 kinds exist, or should it have been raised with them?
- Where are the decision records and review rounds that comments cite ('decision 24', 'decision 9', 'B1', 'Y1-Y5', 'Round 10')? They are not in the repository (docs/ holds only metadata-fields.toml and qa/).

### Coverage

Read in full: `delegates/mail-delegate/Cargo.toml`, `delegates/mail-delegate/src/lib.rs`, `delegates/mail-delegate/src/handlers.rs`, `delegates/mail-delegate/src/store.rs`, `delegates/mail-delegate/src/consent.rs`, `delegates/mail-delegate/src/env.rs`, `delegates/mail-delegate/src/crypto.rs`, `delegates/mail-delegate/src/tests/faults.rs`, `delegates/mail-delegate/src/tests/resilience.rs`, `protocol/src/address.rs`, `ui/src/mail_delegate/local.rs`, `ui/src/mail_delegate/node.rs`, `published-contract/candidate/README.md`, `scripts/check-delegate-candidate.sh`, `scripts/check-release-delegate.sh`, `scripts/delegate-imports-allowlist.txt`, `scripts/check-excluded-crates.sh`, `.release-freeze`

Skimmed: `delegates/mail-delegate/src/tests.rs (module doc, helpers and every test name; bodies of lines 1-447 and 1130-1791 read, 448-1129 by name only)`, `delegates/mail-delegate/src/tests/recovery.rs (module doc and all 21 test names with their doc comments; bodies of lines 1-198 and 690-789)`, `delegates/mail-delegate/src/tests/round2.rs (module doc, all test names; bodies of lines 1-372)`, `delegates/mail-delegate/src/tests/round3.rs (module doc, all test names; bodies of lines 60-160)`, `delegates/mail-delegate/src/watch.rs (lines 1-130 and key constants, for GLOBAL, note_origin, note_seen, run)`, `delegates/mail-delegate/src/jobs.rs (Tick and Trigger only)`, `delegates/mail-delegate/src/tests/jobs.rs (manifest and import-allowlist tests, lines 160-225)`, `protocol/src/lib.rs (lines 1-445 and 536-727)`, `protocol/src/recovery_key.rs (lines 1-130)`, `protocol/src/golden.rs (header and name tables)`, `ui/src/mail_delegate/mod.rs (lines 1-125 and the TTL test)`, `ui/src/aft.rs (lines 470-492)`, `scripts/check-guards-fail.sh (header, helper functions, the guard list for the owned files)`, `scripts/build-wasm.sh (lines 1-80)`, `.github/workflows/build.yml (lines 100-140)`, `Makefile.toml (test and clippy tasks)`, `README.md (lines 30-90)`, `freenet-stdlib 0.12.1 in the cargo registry: delegate_interface.rs (MessageOrigin, InboundDelegateMsg) and delegate_manifest.rs (LifecycleEvent, Capability, wake-up limits)`

Skipped: `delegates/mail-delegate/Cargo.lock: lockfile, skipped as instructed`, `published-contract/candidate/mail_delegate: WASM binary, skipped as instructed (only its size and code hash file were looked at)`
