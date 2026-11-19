> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# v1-frozen: the frozen v0.2.x generation (v1 inbox, Anti-Flood Tokens, old delegates, webapp and container contracts)

Paths are relative to the repo root. Tree read at upstream 2358b59. Nothing was built or run. Short names: INBOX = `contracts/inbox/src/lib.rs`, AFT = `modules/antiflood-tokens/interfaces/src/lib.rs`, REC = `modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs`, GEN = `modules/antiflood-tokens/delegates/token-generator/src/lib.rs`, IDM = `modules/identity-management/src/lib.rs`, MLS = `modules/mail-local-state/src/lib.rs`, WEB = `contracts/web-container/src/lib.rs`, FAC = `contracts/facade/src/lib.rs`.

## 1. Purpose and position

These crates are the source of the WASM that shipped in v0.2.x. The bytes are committed under `published-contract/` and are the keys users' data lives under (AGENTS "Frozen artifacts", correct). Every crate here is outside the workspace with its own lockfile and stdlib `=0.3.5`. The UI never links them: it speaks to the deployed WASM through copies of the types in `wire-v1/`, and `wire-v1/capture` is the only crate that depends on the frozen sources (`wire-v1/capture/Cargo.toml:17-22`).

What the current `use-node` UI still does with this generation:

| Artifact | Runtime use today | Evidence |
|---|---|---|
| v1 inbox contract | Yes: PUT own inbox, GET and subscribe, UPDATE (add, remove, settings). It is still the mail path of every shipped release. | `ui/src/api.rs:768`, `ui/src/inbox.rs:58` |
| token-allocation-record contract | Yes: one per identity, updated on every send | `ui/src/api.rs:1037` |
| token-generator delegate | Yes: registered at first send, prompts the user | `ui/src/api.rs:1180-1182`, `ui/src/aft.rs:474-498` |
| identity-management, mail-local-state delegates | No. Their WASM is embedded only under `cfg(test)` (`ui/src/api.rs:1309-1319`, `ui/src/local_state.rs:731-739`). The migration uses their parameters and registry keys to ask delegates already installed on a v0.2.x user's node (`ui/src/migration/legacy_delegates.rs:84-87`). | |
| webapp contract, container contract | Yes: they serve the app. Publishing exercises them, page code does not. | `published-contract/README.md:21-26` |
| `common/` | Linked by the webapp contract and `tools/web-container-sign` only | `contracts/web-container/Cargo.toml:16` |

The mail delegate also embeds `inbox_code_hash` for its v1 watch (`delegates/mail-delegate/src/handlers.rs:1467`).

## 2. Walkthrough

### contracts/inbox (`freenet-email-inbox`)
- Parameters: `InboxParams { pub_key }`, the owner's ML-DSA-65 verifying key (1952 B), as JSON (INBOX:47-93).
- State: JSON `Inbox { messages, last_update, settings, owner_ek_bytes, inbox_signature }` (INBOX:194-211). `owner_ek_bytes` is the ML-KEM-768 key senders fetch. `inbox_signature` is a private field, so outside code builds the JSON by hand (`contracts/inbox/tests/common/mod.rs:193-234`).
- `InboxSettings { minimum_tier (default Min10), max_age_secs (default 365 d), allow_verified_skip_token, verified_senders: BTreeSet<Vec<u8>>, private }` (INBOX:121-166). `MAX_VERIFIED_SENDERS = 1024` (INBOX:61).
- `Message { content, token_assignment, sender_vk, signature }` (INBOX:179-192).
- Delta: `UpdateInbox::{AddMessages, RemoveMessages{signature, ids}, ModifySettings{inbox_signature, settings, last_update}}`, externally tagged JSON (INBOX:97-119).
- Owner authority: `sign_payload = STATE_UPDATE (8-byte salt, INBOX:33) || owner_ek_bytes || serde_json(settings) || last_update micros i64 BE` (INBOX:383-401). `Inbox::verify` recomputes it (INBOX:444-454). Messages are not under the owner signature.
- `RemoveMessages`: owner signature over the concatenated 32-byte hashes and nothing else (INBOX:693-709).
- `ModifySettings`: cap check, then the signature over the full payload with the state's current ek. Settings, `last_update` and signature are stored verbatim (INBOX:711-737, 906-926).
- Adding a message (INBOX:456-574): skip it if its `assignment_hash` is already present. Otherwise the `TokenAllocationRecord` named by `token_assignment.token_record` must be supplied and `assignment_exists` must hold; `verify_token_policy` needs `tier >= settings.minimum_tier` (INBOX:360); `TokenAssignment::is_valid` checks the slot shape and the ML-DSA signature against the key carried in the assignment itself (INBOX:565-570).
- Verified-sender bypass: with `allow_verified_skip_token` set and `sender_vk` in `verified_senders`, no record is consulted and `verify_sender_signature` over `content` is required instead (INBOX:244-267, 557-563).
- What a sender can do: append. What it cannot: change settings or the ek, or remove (owner signature). On the token path the contract does not check `sender_vk` or `signature` at all; the UI does.
- `validate_state` (INBOX:742-773): owner signature, then `verify_messages` (INBOX:501-544) gives Valid, RequestRelated or Invalid.
- `update_state_impl` (INBOX:838-1009) accepts `State` (verify, then `merge`), `Delta`, `RelatedState` and `RelatedStateAndDelta`; any other variant is an error (INBOX:979-983). Records still missing produce `requires(StateOnce)` (INBOX:987-1008). Every rejection is logged through the host logger (INBOX:786-789).
- `merge` (INBOX:596-634): union of messages by hash through `add_message` (no record lookup). Settings, ek, signature and `last_update` are taken together when `other.last_update` is strictly newer.
- `summarize_state` is the set of hashes; `get_state_delta` is an `AddMessages` of the missing ones (INBOX:793-826). `Inbox::delta` (INBOX:636-649) is unused.
- The contract never advances `last_update` (INBOX:1002-1003); only the owner does.
- Features: `contract` gates `ContractInterface`; `wasmbind` gates `Inbox::new`; the default is `freenet-main-contract` (`contracts/inbox/Cargo.toml:51-55`).

### modules/antiflood-tokens/interfaces (`freenet-aft-interface`)
- `Tier`: `Min1, Min5, Min10, Min30, Hour1, Hour3, Hour6, Hour12, Day1, Day7, Day15, Day30, Day90, Day180, Day365`, `#[repr(u8)]`, serde by variant name, strum Display lowercase (AFT:98-117). `Ord` follows `tier_duration` (AFT:126-136).
- Slots: `is_valid_slot` (AFT:139-193) wants minute tiers on multiples within the hour, hour tiers within the day, and day tiers counted from 31 December of the previous year. `normalize_to_next` (AFT:219-341).
- `TokenAssignment { tier, time_slot, generator, signature, assignment_hash, token_record }`. Signed bytes are `tier u8 || unix seconds i64 LE || hash`, 41 bytes (AFT:741-783). `is_valid` (AFT:793-827). Ordering is by `time_slot` only (AFT:836-840).
- `TokenAllocationRecord { tokens_by_tier: HashMap<Tier, Vec<TokenAssignment>> }`, JSON (AFT:527-615). `assignment_exists` is a binary search on `time_slot` within the tier (AFT:602-610).
- `TokenAllocationSummary(HashMap<Tier, Vec<SummarySlot{time_slot, assignment_hash}>>)` (AFT:683-737).
- `AllocationCriteria { frequency, max_age, contract }`, `max_age <= 730 d` (AFT:489-514).
- Record contract parameters: `TokenDelegateParameters { generator_public_key }`, JSON (AFT:352-393). Generator delegate parameters: `DelegateParameters { generator_seed: [u8;32] }`, JSON (AFT:401-439).
- Delegate messages, bincode: `TokenDelegateMessage::{RequestNewToken, AllocatedToken{delegate_id, assignment, records}, Failure(UserPermissionDenied | NoFreeSlot)}`; `RequestNewToken { request_id, delegate_id, criteria, records, assignment_hash }` (AFT:27-90).

### token-allocation-record contract
- `validate_state`: every assignment `is_valid` under the parameters' key, nothing else (REC:23-40).
- `update_state`: `State` merges; `Delta` is tried as one `TokenAssignment`, then as a whole record (#71); `StateAndDelta`; other variants hit `unreachable!()` (REC:42-149).
- `append` (REC:248-290): the same assignment again is Ok (#72); a different one in an occupied slot is `AllocatedSlot`. This is the only enforcement of one token per slot. Nothing ever removes an assignment.
- `get_state_delta` returns a record-shaped delta, or the whole record on an unreadable summary (REC:160-185).

### token-generator delegate
- `process` ignores `origin` (GEN:41). `RequestNewToken` is stashed as secret `pending_request_<id LE>` (GEN:30-34, 67-70) and answered with `RequestUserInput { request_id, responses ["true","false"], message }` plus an echo of the request (GEN:162-194).
- The prompt is JSON `{"user":"ml-dsa-65:<16 bytes hex>...","token":{"tier":..,"max_age":"N seconds"}}` (GEN:127-140). The delegate remembers no grant: every request prompts.
- `UserResponse` reads and removes the stash. `Allowed` gives `AllocatedToken` (`processed = true`) or `Failure(NoFreeSlot)`; `NotAllowed` gives `Failure(UserPermissionDenied)` (GEN:81-119, 212-252).
- `assign` (GEN:321-359) takes the host clock on wasm and signs the first free slot from `next_free_assignment` (GEN:361-421): scan upward from `normalize_to_next(now) - max_age`, `None` when the closed window is full. Allocation is oldest first, so a new record can spend the whole window at once (52,561 Min10 slots at 365 d).

### identity-management and mail-local-state delegates
- Parameters are a P-384 secret key as JSON (`IdentityParams`, IDM:12-69; `LocalStateParams`, MLS:16-68). Each delegate stores one secret whose key is those parameter bytes (IDM:167-169; MLS:721-723).
- IDM value: JSON `IdentityManagement { identities: HashMap<Alias, AliasInfo> }`, with `AliasInfo { key, extra, kind, inbox_wasm_hash, pending_migration_from, aft_record_wasm_hash, pending_aft_migration_from }` (IDM:85-131). `IdentityMsg` (JSON): `Init, CreateIdentity, DeleteIdentity, UpdateInboxWasmHash, SetPendingMigrationFrom, UpdateAftRecordWasmHash, SetPendingAftMigrationFrom, GetIdentities` (IDM:366-419).
- MLS value: JSON `LocalState { aliases: HashMap<Alias, AliasState>, settings: GlobalSettings }` (MLS:521-529), with `AliasState { drafts, read, kept, sent, archived, deleted, settings }` (MLS:359-383). `LocalStateMsg` (JSON): `Init, SaveDraft, DeleteDraft, MarkRead, SaveSent, DeleteSent, SetSentDeliveryState, ArchiveMessage, UnarchiveMessage, DeleteMessage, SetIdentitySettings, SetGlobalSettings, GetAll` (MLS:615-695).
- Both: `origin` ignored, no prompt, mutators return no message. Only `GetIdentities` and `GetAll` answer, with the raw stored JSON (IDM:345-358; MLS:858-865). Each call rewrites the whole blob (MLS:873-884).
- `src/bin/tool.rs` in each writes a key pair and `<name>-params`.

### contracts/web-container (webapp contract) and common/
- `common/src/web_container.rs:4-8`: `WebContainerMetadata { version: u32, signature }`. `common/src/facade.rs` re-exports the facade types.
- State: `[meta_len u64 BE][CBOR metadata][web_len u64 BE][web]`; signed payload `version u32 BE || web` (WEB:8-9, 86-92). Parameters: exactly 32 bytes, the ed25519 key (WEB:31-41). Caps: metadata 1 KiB, web 100 MiB (WEB:18-19). `version == 0` is invalid (WEB:65-67).
- `update_state` takes the first item only, accepts `UpdateData::State` only, and compares versions (strictly greater). It checks no signature (WEB:97-124).

### contracts/facade, facade-types, facade-loader (container contract)
- `FacadePointer { version: u64, current_app_id: [u8;32], prev_app_ids: Vec<(u64,[u8;32])> }`, `FACADE_MAX_PREV_APP_IDS = 3`, `signed_payload = version u64 BE || app id || count u32 BE || (version || id)* || loader bytes` (`contracts/facade-types/src/lib.rs:18-65`).
- Same framing as the webapp contract; caps 4 KiB and 256 KiB (FAC:52-53). `verify_full` (FAC:144-194): version non-zero, ring cap, every previous version below the current one and strictly descending, current id not in the ring, `verify_strict`. `parse_state` refuses trailing bytes (FAC:242-247).
- `update_state`: exactly one `UpdateData::State`, full verification of the new state, then a strictly greater version (FAC:68-108).
- Loader: `contracts/facade-loader/src/index.html.tmpl:31` holds `__CURRENT_APP_ID__`. A framed page posts `{__freenet_shell__, type:'navigate', href}` to its parent; a standalone page uses `location.replace` (`:57-70`). AGENTS describes this correctly.
- Toolchain pin 1.95.0 (`contracts/facade/rust-toolchain.toml:17`).

### Registries and configuration
- `legacy_inbox.toml:17-25`: generations 0 (`Ht8rc...`) and 205 (`9Kuhod...`). The two delegate registries each list 205 and 206 with `delegate_key` and `params_hex`; 206 is the hash frozen in `published-contract/wasm/v1/`. Their consumers are mapped in `ui-migration.md`.
- `contracts/.cargo/config.toml` and `modules/.cargo/config.toml` set `--cfg getrandom_backend="custom"` for wasm32 builds started inside those directories. The first also covers `contracts/inbox-v2`.

## 3. Interfaces

- Encodings: inbox, record and all parameters are `serde_json`, with byte vectors as JSON number arrays, so one token costs roughly 20 KB of JSON. Generator messages and context are bincode. Webapp and container metadata are CBOR inside big-endian length framing.
- Keys: inbox id = hash(inbox wasm, JSON `InboxParams`). Record id = hash(record wasm, JSON `TokenDelegateParameters`) with the identity's own verifying key (`ui/src/aft.rs:507-512`). Generator delegate key = `DelegateKey::from_params(code hash, JSON DelegateParameters)` (`ui/src/api.rs:1230`).
- The generator's parameters are the identity's ML-DSA seed. The page gets them from the mail delegate's `V1AftParams`, which exists only with the `v1-transition` feature (`ui/src/aft.rs:474-498`). A delegate built without that feature cannot send v1 mail.
- The UI sends a message as `RelatedStateAndDelta` with its own record state inline (`ui/src/inbox.rs:651-667`; INBOX:932-943), and uses blake3 of the sealed content as `assignment_hash` (`ui/src/inbox.rs:760`).
- Host APIs used: `freenet_stdlib::log::info`, `time::now` (generator), `DelegateCtx::{get_secret, set_secret, remove_secret}`, `RequestUserInput`. `tracing` in REC comes from the stdlib prelude.
- Callers in: `wire-v1/capture` (golden bytes), `tools/web-container-sign` (through `common/`), `scripts/check-wasm-builds.sh`, `scripts/check-container-byte-equal.sh`.

## 4. Invariants

| Rule | Enforced | Pinned by |
|---|---|---|
| Shipped bytes never change | `published-contract/frozen.sha256`, `scripts/check-frozen-artifacts.sh`, pre-commit hook | CI `check-drift` |
| Serialized shapes never change, including the field order of `InboxSettings` (its JSON is signed, INBOX:390-391) | convention in these sources | `wire-v1/src/golden_v1.txt` through `cargo make check-golden-capture` (`Makefile.toml:949-960`) |
| `STATE_UPDATE` salt, `Tier` discriminants (signed as `tier as u8`, AFT:776) and variant names | constants | golden file; `tier_ord_follows_slot_duration` (AFT:988) for order only |
| Owner signature covers ek, settings and `last_update` | INBOX:444-454, 884 | `validate_rejects_*`, `update_state_arm_rejects_*` |
| Settings are last-write-wins on the signed `last_update` | INBOX:623-632 | `state_broadcast_with_newer_*`, `_older_*` |
| Tier is a floor, not an equality | INBOX:360 | `update_accepts_stronger_tier_*`, `update_rejects_token_with_weaker_tier_*` |
| Bypass needs the sender's signature | INBOX:557-563 | `bypass_rejects_*` |
| `verified_senders <= 1024` | INBOX:720-724 | `modify_settings_rejects_oversized_*`, `_accepts_*_at_cap` |
| A replay of a present message is dropped | INBOX:468-476 | `update_dedups_replay_*`, `bypass_message_replay_*` |
| One assignment per slot; an identical re-send is Ok | REC:263-283 | `append_rejects_distinct_*`, `append_is_idempotent_*` |
| Webapp: version non-zero and strictly rising; signature over version and web | WEB:65-67, 111-118, 86-92 | six unit tests (WEB:209-316) |
| Container: full verification on update, ring rules, no trailing bytes | FAC:88, 165-186, 242-247 | fifteen unit tests (FAC:297-566) |
| Old delegate parameters are part of the delegate key | registry `delegate_key` rows | `ui/build.rs`, `ui/src/legacy_registry.rs` |
| Container toolchain stays 1.95.0 | `contracts/facade/rust-toolchain.toml` | `scripts/check-container-byte-equal.sh` |

## 5. State

- Frozen and shipped: everything in `published-contract/` except `candidate/` (18 pinned files in `frozen.sha256`). Code hashes: inbox `Av3KF7WS...`, record `4cLK4421...`, generator `H4kgUiEd...`, identity `BbtKhHhR...`, local state `5FGNJrAW...`.
- Nothing in these paths is a candidate.
- The sources are kept readable and buildable, not reproducible: `scripts/check-wasm-builds.sh:9-10` says its output "need not match the frozen v1 bytes". No check ties the source in this tree to the shipped WASM.
- End of life is planned, not done: the v1 send path goes when `v1-transition` goes (`ui/src/aft.rs:476-477`).

## 6. Tests

All are native and need no node. Prerequisite: network access for each crate's lockfile on first run.

| Where | Count | Proves | Run |
|---|---|---|---|
| `contracts/inbox/tests/integration.rs`, helpers in `tests/common/mod.rs` | 40 | Calls `Inbox::validate_state`, `update_state`, `summarize_state`, `get_state_delta` directly with fresh ML-DSA keys: owner-signature binding, token add paths, tier floor, dedup, the bypass matrix, the settings cap, LWW, #253 tampering | `cargo make test-inbox` (= `cargo test --features contract --test integration`, `Makefile.toml:926-933`) |
| INBOX:1012-1156 | 3 | Wire shape of `UpdateInbox`, the `max_age_secs` default, an empty-inbox validation | No task runs them: they need `--features contract` without `--test` |
| AFT:865-1272 | 19 | Slot validity and normalisation samples, tier order, decode errors, JSON round trips, summary lookup by hash | `scripts/check-excluded-crates.sh test` (part of `cargo make test`) |
| REC:293-437 | 3 | Idempotent append, slot collision, delta against a summary | same script |
| REC `tests/test.rs` | 1, ignored | RSA-era fixture; needs a file no step creates (`tests/test.rs:6-11`) | never |
| GEN `src/tests.rs` | 7 | `next_free_assignment` for Day1 only | same script |
| IDM:438-683, MLS:886-1359 | 16, 31 | Serde defaults and round trips; `process` is never called | same script |
| WEB, FAC | 6, 15 | Section 4 | WEB by the script; FAC only by `cd contracts/facade && cargo test` (CI `build.yml:138-144`) |

Not covered in these crates: `validate_state` over a state that holds messages (every call passes `RelatedContracts::new()` and no messages), `RemoveMessages`, the `RelatedStateAndDelta` arm, any allocator tier other than Day1, the generator's prompt round trip, and the handlers of both old delegates.

## 7. Changing it safely

- A source edit here changes nothing that ships (AGENTS, correct). It can still break `check-golden-capture` (a serialized shape moved), `check-wasm-builds.sh`, clippy and tests of the excluded crates, and, for `contracts/facade`, the byte-equality gate, which rebuilds that source.
- `common/` is in the workspace and is a path dependency of the webapp contract: a change there can stale `contracts/web-container/Cargo.lock`, and `--locked` then fails.
- A contract or delegate fix is a new generation: registry row, `scripts/build-wasm.sh`, `published-contract/wasm/v<N>/`, `frozen.sha256`, the UI includes, `ui/build.rs:51-74`, and a data migration (`published-contract/README.md:78-99`). For the inbox, also the in-source slice at `ui/src/inbox.rs:71-76`.
- Mirror every type change in `wire-v1/` and add golden samples through `wire-v1/capture`.
- Never rotate the old delegates' parameter files or the committed test keys behind them.
- Leave the dependencies, profile and toolchain of `contracts/facade` alone unless re-keying the bookmarked URL is the goal.
- Traps: the inbox `contract` feature is off by default, so a plain `cargo test` in `contracts/inbox` runs nothing. `contracts/facade` is not in `scripts/check-excluded-crates.sh:12-23`. The `getrandom` cfg applies by working directory, not by crate.

## 8. Disagreements between documents and code

1. AGENTS says the token generator's key "derives from the identity's ML-DSA seed, not from the WASM". It is `from_params(code hash, params)` (`ui/src/api.rs:1230`), so it depends on both.
2. AGENTS and `common/src/facade.rs:6-8` say the UI imports the facade types through `freenet-email-core`. The UI has no such dependency.
3. `modules/antiflood-tokens/interfaces/Cargo.toml:19-21` says the UI consumes that crate through `common/`. The UI uses `wire-v1`'s copy, and `common/` re-exports no AFT types.
4. `contracts/inbox/Cargo.toml:1-4` says `contracts/facade/Cargo.toml` uses the same `[workspace]` stanza, and the interfaces manifest (`:28-31`) says every excluded crate has one. `contracts/facade` and `contracts/facade-types` have none.
5. INBOX:123-127 says tokens must be minted "at this exact tier", and INBOX:219-221 speaks of `InboxParams.required_tier`. The check is a floor on a settings field (INBOX:360).
6. INBOX:614-627 and `contracts/inbox/tests/common/mod.rs:207-209` describe the signed payload as `STATE_UPDATE || owner_ek_bytes`. It has four parts (INBOX:383-401).
7. `contracts/inbox/tests/common/mod.rs:58-60` says the default policy is `Tier::Day1`. It is `Min10` and is not in the parameters.
8. AFT:755 calls `assignment_hash` a Blake2s256 hash. The UI uses blake3 (`ui/src/inbox.rs:760`), and the contract never checks it.
9. `published-contract/README.md:40-43` says the UI embeds all five artifacts. The two old delegates' WASM is embedded only in tests.
10. AGENTS "Regenerating test keys" counts two committed test keys. There are three sets (`test-contract/mail-local-state/` as well).
11. INBOX:5 names `cargo make build-inbox`. The task is `build-inbox-contract` (`Makefile.toml:28`).
12. `contracts/facade-loader/README.md:19-20` says `build-container` embeds `dist/`. The Makefile follows AGENTS' lifecycle: pack the loader, then sign the state.
13. `legacy_inbox.toml` has two rows and the slice the contract migration walks has one; the registry headers say `legacy_registry.rs` "refuses" a registry naming the generation in use, which is a unit test. Both are detailed in `ui-migration.md`.

## 9. Suspected bugs and weak spots

All are in frozen code, so none can be patched in place.

1. A token is not bound to its record (high confidence from the code). `assignment_exists` matches tier and slot only (AFT:602-610), and `is_valid` trusts the key inside the assignment (INBOX:565-570; the same in INBOX:525-538). Any key can sign a token for a slot that exists in any record, and one burned slot can back any number of messages with different hashes.
2. `update_state` trusts caller-supplied related state (INBOX:928-943). Only `validate_state` on the outcome, with records the host fetches, checks the real record (medium; depends on the host).
3. `verify_messages` skips bypass messages without checking the sender signature (INBOX:510-515), while `add_message` checks it (INBOX:557-563). A first PUT of a state is held only to the weaker rule (medium).
4. Stored messages are re-judged against current settings. Raising `minimum_tier`, switching the bypass off or removing a verified sender leaves messages that fail the policy or name a record that does not exist (INBOX:516-532). `toggle_bypass_off_*` never validates the state it produces (`contracts/inbox/tests/integration.rs:897-980`) (medium-high).
5. No tombstones: a removed message returns through `merge` or a replayed `AddMessages` (INBOX:468-476, 604-613) (high).
6. `content` is not bound to the token: the contract never hashes it, and dedup is first-wins (INBOX:473-476, 546-574) (medium).
7. `Tier::normalize_to_next`: Min30 uses base 15 (AFT:233) and gives `:15` or `:45` slots, which `is_valid_slot` rejects, for half of each hour (high). Day tiers of 7 and more call `with_day(day - remainder).unwrap()` (AFT:336), which panics when the remainder reaches the day of the month; `next_slot` and `normalized - max_age` also ignore the per-year anchor (high for the panic). Only Day1 is tested.
8. REC: the first assignment of a tier is inserted without `is_valid` (REC:285-288); `validate_state` checks neither slot uniqueness nor order (REC:23-40); other `UpdateData` variants panic (REC:144); records only grow.
9. WEB: `read_version` allocates `metadata_size` bytes with no cap (WEB:164-167); trailing bytes are accepted; `update_state` relies on outcome validation for the signature. WEB and FAC both decode the summary before use, so an empty summary is an error (WEB:150-151; FAC:132-133; medium), and both reject the deltas they produce (WEB:108-123; FAC:78-80).
10. The old delegates ignore `set_secret`'s result (IDM:203; MLS:882) and answer nothing, so a failed write is silent. Any app that knows the public parameters can read everything (IDM:164, 345-358).
11. Generator: a prompt never answered leaves its stash secret for good (GEN:67-70).
12. Stale test: `validate_test` signs only `STATE_UPDATE` (INBOX:1043) and would fail under the current `verify`; no task runs it. `contracts/inbox/examples/initial_state.json` carries a 512-byte signature.
13. The webapp and container payloads share one key with no domain tag. Reuse looks blocked by the version-zero check and the 256 KiB loader cap (low).

## 10. Open work

- INBOX:860: "fixme: take care of race condition between token alloc record and the assignment".
- INBOX:1002-1003: `last_update` is not advanced by the contract.
- GEN:368: allocator optimisation todo.
- REC `tests/test.rs:6-11`: ignored test awaiting a fixture.
- `contracts/facade-loader/README.md:32-36`: phase 2 loader (subscribe, automatic rollback) deferred.
- MLS:84-86: sent rows that stay `Pending` (#58). MLS:96-99: unarchive (#60).
- `modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp`: a disabled runtime test.

## 11. Questions the tree does not answer

1. Does the host always run `validate_state`, with related contracts it fetched itself, on the outcome of `update_state`? Items 2, 8 and 9 of section 9 depend on it.
2. What does the stdlib 0.3.5 host do with the generator's echoed `RequestNewToken` (`processed = false`, GEN:181-193)?
3. Were the shipped v1 bytes built from exactly this source? Nothing checks it.
4. Do hosts deliver `get_state_delta` output to peers as `UpdateData::Delta` for the webapp and container contracts?
5. Is any recipient using Min30 or a day tier of 7 or more in production?

---

## Appendix: the reader's structured summary

The frozen v0.2.x generation: source for the WASM committed under published-contract/ (v1 inbox contract, token-allocation-record contract, token-generator delegate, identity-management and mail-local-state delegates, webapp contract, container/facade contract) plus common/ and the three legacy registries. All crates sit outside the workspace on stdlib 0.3.5; the UI links none of them and talks to the deployed WASM through wire-v1 copies.

Still used at runtime by the use-node UI: the v1 inbox contract, the token record contract and the token-generator delegate (every send prompts the user), and the webapp and container contracts that serve the app. Migration only: the two old delegates, whose WASM is embedded only in tests; the walk uses their parameters and registry keys.

State: shipped and frozen, no candidate here, end of life tied to removing the delegate's v1-transition feature. The sources are kept buildable but nothing proves they match the shipped bytes. Tests are native unit and integration tests; coverage has real holes (validate_state over messages, RemoveMessages, RelatedStateAndDelta, non-Day1 tiers, all delegate handlers), and three inbox unit tests are run by no task, one of which would fail.

Reading the code turned up weaknesses that cannot be patched in place: the inbox does not bind a token to its record or its content, removals have no tombstones, Min30 and day tiers of 7+ compute invalid slots or panic. Twelve-plus comment and doc statements disagree with the code.

The map file is complete on disk (3,333 words), so a restart of this unit can reuse it rather than redo it.

### Key facts

- Runtime vs migration: the UI still embeds and uses the v1 inbox, token record and token generator WASM (ui/src/api.rs:768, ui/src/api.rs:1037, ui/src/api.rs:1180-1182); the identity-management and mail-local-state WASM is embedded only under cfg(test) (ui/src/api.rs:1309-1319, ui/src/local_state.rs:731-739) and only their params are used by the migration (ui/src/migration/legacy_delegates.rs:84-87).
- No UI crate depends on the frozen sources; wire-v1/capture is the only dependent (wire-v1/capture/Cargo.toml:17-22), and cargo make check-golden-capture diffs its output against wire-v1/src/golden_v1.txt (Makefile.toml:949-960).
- Inbox params are JSON InboxParams{pub_key} (owner ML-DSA-65 VK); state is JSON Inbox{messages,last_update,settings,owner_ek_bytes,inbox_signature} (contracts/inbox/src/lib.rs:47-93, 194-211).
- Owner signature payload = STATE_UPDATE(8 B) || owner_ek_bytes || serde_json(settings) || last_update micros i64 BE; messages are not covered (contracts/inbox/src/lib.rs:33, 383-401, 444-454).
- Inbox deltas: UpdateInbox::{AddMessages, RemoveMessages{signature over concatenated hashes}, ModifySettings{inbox_signature, settings, last_update}} as externally tagged JSON (contracts/inbox/src/lib.rs:97-119, 693-737).
- Token check on add: dedup by assignment_hash, record must contain the (tier, slot), tier >= settings.minimum_tier, signature valid under the key carried in the assignment itself (contracts/inbox/src/lib.rs:456-574, 360; modules/antiflood-tokens/interfaces/src/lib.rs:602-610).
- Verified-sender bypass skips the token and requires an ML-DSA signature over content; verified_senders is capped at 1024 (contracts/inbox/src/lib.rs:61, 244-267, 557-563, 720-724).
- update_state accepts State, Delta, RelatedState, RelatedStateAndDelta only; merge unions messages and takes settings+ek+signature when last_update is strictly newer (contracts/inbox/src/lib.rs:838-1009, 596-634). The UI sends with RelatedStateAndDelta (ui/src/inbox.rs:651-667).
- Tier has 15 variants Min1..Day365, repr(u8), signed as tier u8 || unix secs i64 LE || 32-byte hash (41 B); Ord follows duration (modules/antiflood-tokens/interfaces/src/lib.rs:98-136, 768-783).
- The record contract's append is the only one-token-per-slot enforcement; identical re-send is idempotent; nothing removes assignments (modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs:248-290).
- Token generator: params are DelegateParameters{generator_seed}; every RequestNewToken prompts via RequestUserInput with responses "true"/"false"; request stashed as secret pending_request_<id>; origin ignored (modules/antiflood-tokens/delegates/token-generator/src/lib.rs:30-34, 38-122, 125-140).
- The generator seed is the identity's ML-DSA seed, handed to the page by the mail delegate's V1AftParams, which exists only with the v1-transition feature; without it v1 sending is impossible (ui/src/aft.rs:474-498).
- Allocation is oldest-first from normalize_to_next(now) - max_age; None when the closed window is full; max_age capped at 730 d (modules/antiflood-tokens/delegates/token-generator/src/lib.rs:361-421; modules/antiflood-tokens/interfaces/src/lib.rs:504).
- Old delegates: params are a P-384 secret key as JSON; one secret per delegate keyed by the params bytes holding one JSON blob; no origin gate, no prompt; only GetIdentities/GetAll answer (modules/identity-management/src/lib.rs:12-69, 159-364; modules/mail-local-state/src/lib.rs:16-68, 713-884).
- Webapp contract: state [meta_len u64 BE][CBOR WebContainerMetadata{version u32, signature}][web_len][web], payload version u32 BE || web, params 32-byte ed25519 key; update_state only compares versions (contracts/web-container/src/lib.rs:8-9, 31-41, 86-92, 97-124; common/src/web_container.rs:4-8).
- Container contract: FacadePointer{version u64, current_app_id, prev_app_ids (max 3)}; payload version || id || count || (version||id)* || loader; update runs full verification then strictly greater version; no trailing bytes (contracts/facade-types/src/lib.rs:18-65; contracts/facade/src/lib.rs:68-108, 144-194, 242-247).
- contracts/facade and contracts/facade-types are not in scripts/check-excluded-crates.sh:12-23, so cargo make test and clippy skip them; only CI build.yml:138-144 covers the facade.
- cargo make test-inbox runs only the integration target (Makefile.toml:926-933); the default cargo test in contracts/inbox compiles no tests because the contract feature is off (contracts/inbox/Cargo.toml:51-55; contracts/inbox/tests/integration.rs:9).
- scripts/check-wasm-builds.sh builds the five v1 sources and discards the output, stating it need not match the frozen bytes (scripts/check-wasm-builds.sh:9-10); no check ties this source to the shipped WASM.
- Registries: legacy_inbox.toml has generations 0 and 205 (legacy_inbox.toml:17-25); each delegate registry has 205 and 206, 206 being the hash frozen in published-contract/wasm/v1 (legacy_identity_delegate.toml:20-32, legacy_local_state_delegate.toml:19-31).

### Docs or comments that disagree with the code

- **[medium]** Claim (AGENTS.md, 'AFT contract migration (#251 improvement 3)' paragraph): The token-generator delegate's key derives from the identity's ML-DSA seed, not from the WASM, so it never rotates under a contract bump. **Reality:** The key is DelegateKey::from_params(TOKEN_GEN_CODE_HASH, params); it depends on the WASM hash and the seed. A new generator WASM is a new delegate key (it holds only pending-request secrets, so nothing needs migrating). Evidence: ui/src/api.rs:1230
- **[low]** Claim (AGENTS.md 'Container contract lockfile isolation'; common/src/facade.rs:6-8): The workspace re-exports the facade types through freenet-email-core so the signer and the UI keep their imports. **Reality:** Only tools/web-container-sign and contracts/web-container depend on freenet-email-core. The UI has no such dependency. Evidence: tools/web-container-sign/Cargo.toml:14
- **[low]** Claim (modules/antiflood-tokens/interfaces/Cargo.toml:19-21): The UI also consumes freenet-aft-interface, via re-export through common/. **Reality:** The UI uses wire-v1's copy of the types; common/ re-exports only facade types and WebContainerMetadata. Evidence: common/src/lib.rs:1-3
- **[low]** Claim (contracts/inbox/Cargo.toml:1-4; modules/antiflood-tokens/interfaces/Cargo.toml:28-31): contracts/facade/Cargo.toml uses the same [workspace] stanza; every excluded crate is its own workspace root. **Reality:** contracts/facade and contracts/facade-types have no [workspace] stanza (facade-types also has no Cargo.lock). Evidence: contracts/facade/Cargo.toml:1-6
- **[medium]** Claim (contracts/inbox/src/lib.rs:123-127 and :219-221 (code comments)): Senders must mint tokens at this exact tier; the tier policy is InboxParams.required_tier authenticated by hash(code, params). **Reality:** The policy is InboxSettings.minimum_tier, mutable, and the check is a floor: assignment.tier < minimum_tier is rejected. Evidence: contracts/inbox/src/lib.rs:360
- **[low]** Claim (contracts/inbox/src/lib.rs:614-627 (merge comment); contracts/inbox/tests/common/mod.rs:207-209): The owner signs STATE_UPDATE || owner_ek_bytes. **Reality:** The payload is STATE_UPDATE || ek || serde_json(settings) || last_update micros. Evidence: contracts/inbox/src/lib.rs:383-401
- **[low]** Claim (contracts/inbox/tests/common/mod.rs:58-60): make_params uses the default recipient policy (Tier::Day1, 365d). **Reality:** Params carry only the key; the default settings tier is Min10. Evidence: contracts/inbox/src/lib.rs:156-166
- **[low]** Claim (modules/antiflood-tokens/interfaces/src/lib.rs:755): assignment_hash is a Blake2s256 hash of the message. **Reality:** The UI uses blake3 of the sealed content; no contract checks the hash against the content. Evidence: ui/src/inbox.rs:760
- **[low]** Claim (published-contract/README.md:40-43): The UI pulls in all five wasm/v1 artifacts (api.rs, inbox.rs, aft.rs, local_state.rs). **Reality:** identity_management and mail_local_state WASM are included only in cfg(test) modules; the runtime uses their params only. Evidence: ui/src/api.rs:1309-1319
- **[low]** Claim (AGENTS.md 'Regenerating test keys'): Two committed test keys live under test-contract/. **Reality:** There are three sets: web-container-keys.toml, identity/, and mail-local-state/ (which legacy_local_state_delegate.toml:12-13 refers to). Evidence: legacy_local_state_delegate.toml:12
- **[low]** Claim (contracts/inbox/src/lib.rs:5): cargo make build-inbox builds the inbox with the contract feature. **Reality:** The task is build-inbox-contract. Evidence: Makefile.toml:28
- **[low]** Claim (contracts/facade-loader/README.md:19-20): cargo make build-container embeds dist/ as the container contract's webapp content. **Reality:** The loader is packed (pack-container-loader) and signed into the state (sign-container-state); build-container builds the WASM. Evidence: Makefile.toml:252
- **[medium]** Claim (AGENTS.md 'Chained migration'; legacy_inbox.toml:6-7): Append a bumped inbox hash to the slice and to legacy_inbox.toml (the lists agree); legacy_registry.rs refuses a registry containing the generation in use. **Reality:** The registry has two rows, the in-source slice one (9Kuhod... is missing from the slice); the refusal is a unit test, not a build check. Detailed in ui-migration.md. Evidence: ui/src/inbox.rs:71-76
- **[low]** Claim (AGENTS.md 'Testing'; Makefile.toml:908): cargo make test and clippy cover every out-of-workspace crate. **Reality:** contracts/facade and contracts/facade-types are not in the script's list; the facade is tested only by a CI step. Evidence: scripts/check-excluded-crates.sh:12-23
- **[low]** Claim (contracts/inbox/tests/common/mod.rs:195-197): make_inbox_state's format is verified by the existing validate_test unit test. **Reality:** validate_test signs only STATE_UPDATE and no task runs it. Evidence: contracts/inbox/src/lib.rs:1043

### Suspected issues (leads from reading, not confirmed)

- **v1 inbox does not bind a token to its record or generator** (confidence: high). assignment_exists matches tier and time_slot only, and is_valid verifies against the key carried inside the assignment. Any key can sign a token for a slot present in any record, and one burned slot backs unlimited messages with different hashes. Applies on both the update and validate paths. Frozen; cannot be patched. Evidence: modules/antiflood-tokens/interfaces/src/lib.rs:602-610; contracts/inbox/src/lib.rs:565-570
- **update_state trusts caller-supplied related state** (confidence: medium). RelatedState and RelatedStateAndDelta records are accepted as the record for related_to with no check; only validate_state on the outcome, with host-fetched records, sees the real record. Evidence: contracts/inbox/src/lib.rs:928-943
- **validate_state skips the sender signature on bypass messages** (confidence: medium). verify_messages continues past bypass messages without verify_sender_signature, while add_message requires it. A state first seen through PUT/validate is held to the weaker rule. Evidence: contracts/inbox/src/lib.rs:510-515
- **Settings changes can make stored messages invalid** (confidence: medium). verify_messages re-judges stored messages against current settings. Raising minimum_tier, turning the bypass off or removing a verified sender leaves messages that fail the tier floor or point at a record that does not exist. The toggle test never validates the resulting state. Evidence: contracts/inbox/src/lib.rs:516-532; contracts/inbox/tests/integration.rs:897-980
- **Removed messages can come back** (confidence: high). No tombstones: the replay guard only covers messages still present, and merge unions any message a peer still holds. Evidence: contracts/inbox/src/lib.rs:468-476, 604-613
- **Message content is not bound to its token** (confidence: medium). The contract never hashes content against assignment_hash, and dedup keeps the first message with a hash, so a relayer could substitute content under a valid token. Evidence: contracts/inbox/src/lib.rs:473-476, 546-574
- **Min30 normalisation uses base 15** (confidence: high). normalize_to_next for Min30 calls normalize_to_next_minute(time, 15), yielding :15 or :45 slots for half of each hour; is_valid_slot requires a multiple of 30, so the token is rejected. Evidence: modules/antiflood-tokens/interfaces/src/lib.rs:233
- **Day tiers of 7+ panic or misalign** (confidence: high). normalize_to_next_day does with_day(day - remainder).unwrap(), which is None (or underflows) when the remainder reaches the day of month; next_slot and normalized - max_age ignore the per-year anchor is_valid_slot uses. Only Day1 is tested in the allocator. Evidence: modules/antiflood-tokens/interfaces/src/lib.rs:336
- **Record contract gaps** (confidence: medium). First assignment of a tier is inserted without is_valid; validate_state checks neither slot uniqueness nor sort order (assignment_exists binary-searches); unknown UpdateData variants hit unreachable!(); records never shrink and each assignment is about 20 KB of JSON. Evidence: modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs:285-288, 23-40, 144
- **Webapp contract: uncapped allocation and no signature check in update_state** (confidence: medium). read_version allocates metadata_size bytes without the 1 KiB cap; update_state compares versions only and relies on outcome validation; trailing bytes are accepted. Evidence: contracts/web-container/src/lib.rs:164-167, 97-124
- **Webapp and container contracts reject their own deltas and error on an empty summary** (confidence: medium). get_state_delta returns the full state as a delta, but update_state accepts only UpdateData::State; the summary is CBOR-decoded before use, so an empty summary is a Deser error. Evidence: contracts/web-container/src/lib.rs:108-123, 150-151; contracts/facade/src/lib.rs:78-80, 132-133
- **Old delegates lose writes silently and have no access control** (confidence: high). set_secret's bool result is ignored and mutators send no reply; origin is ignored and the params are public, so any app that can address the delegate can read every identity key and all local state. Evidence: modules/identity-management/src/lib.rs:164, 203, 345-358; modules/mail-local-state/src/lib.rs:880-884
- **Stale inbox unit test that no task runs** (confidence: high). validate_test signs only STATE_UPDATE and would fail under the current verify; it and two other lib tests need --features contract without --test, which no Makefile task or script uses. examples/initial_state.json holds a 512-byte RSA-era signature. Evidence: contracts/inbox/src/lib.rs:1012-1072; Makefile.toml:926-933
- **Unanswered token prompts leak a stash secret** (confidence: medium). The pending request secret is removed only on UserResponse. Evidence: modules/antiflood-tokens/delegates/token-generator/src/lib.rs:67-70, 114
- **No domain separation between webapp and container signatures** (confidence: low). Both payloads are signed by the same production key with no tag; reuse appears blocked by the version-zero check and the 256 KiB loader cap. Evidence: contracts/web-container/src/lib.rs:86-92; contracts/facade-types/src/lib.rs:53-65

### Open work

- contracts/inbox/src/lib.rs:860 fixme: race between the token allocation record and the assignment
- contracts/inbox/src/lib.rs:1002-1003 FIXME: last_update is not advanced by the contract
- modules/antiflood-tokens/delegates/token-generator/src/lib.rs:368 todo: allocator optimisation when all slots are taken
- modules/antiflood-tokens/contracts/token-allocation-record/tests/test.rs:6-11 ignored test waiting for a fixture no build step produces (RSA-era bytes)
- modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp disabled runtime test
- contracts/facade-loader/README.md:32-36 phase 2 loader (subscribe to pointer, automatic rollback to prev_app_ids) deferred
- modules/mail-local-state/src/lib.rs:84-86 sent rows that stay Pending (#58); :96-99 true unarchive (#60)
- ui/src/aft.rs:476-477 the v1 send path (V1AftParams, token generator) is to be removed with the anti-flood tokens
- Untested here: validate_state over stored messages, RemoveMessages, RelatedStateAndDelta arm, allocator tiers other than Day1, generator prompt round trip, both old delegates' process handlers (contracts/inbox/tests/integration.rs:38-165 shows every validate call uses empty related contracts)

### Open questions

- Does the host always run validate_state, with related contracts it fetched itself, on the outcome of update_state? Several weak spots (caller-supplied related state, record append without is_valid, webapp update without signature check) are only contained if it does.
- What does the stdlib 0.3.5 host do with the token generator's echoed RequestNewToken ApplicationMessage that is not marked processed (token-generator/src/lib.rs:181-193)?
- Were the shipped v1 bytes built from exactly the source in this tree? check-wasm-builds.sh discards its output and nothing else compares.
- Do nodes deliver get_state_delta output to peers as UpdateData::Delta for the webapp and container contracts, which both reject deltas?
- Is any recipient in production using Min30 or a day tier of 7 or more, where the frozen generator computes invalid slots or panics?
- Is an empty state summary ever passed to get_state_delta of the webapp or container contract (it would be a decode error)?

### Coverage

Read in full: `contracts/inbox/src/lib.rs`, `contracts/inbox/Cargo.toml`, `contracts/inbox/freenet.toml`, `contracts/inbox/examples/initial_state.json`, `contracts/inbox/tests/common/mod.rs`, `modules/antiflood-tokens/interfaces/src/lib.rs`, `modules/antiflood-tokens/interfaces/Cargo.toml`, `modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs`, `modules/antiflood-tokens/contracts/token-allocation-record/Cargo.toml`, `modules/antiflood-tokens/contracts/token-allocation-record/freenet.toml`, `modules/antiflood-tokens/contracts/token-allocation-record/tests/test.rs`, `modules/antiflood-tokens/delegates/token-generator/src/lib.rs`, `modules/antiflood-tokens/delegates/token-generator/src/tests.rs`, `modules/antiflood-tokens/delegates/token-generator/Cargo.toml`, `modules/identity-management/src/lib.rs`, `modules/identity-management/src/bin/tool.rs`, `modules/identity-management/Cargo.toml`, `modules/mail-local-state/Cargo.toml`, `contracts/web-container/src/lib.rs`, `contracts/web-container/Cargo.toml`, `contracts/web-container/freenet.toml`, `contracts/facade/src/lib.rs`, `contracts/facade/Cargo.toml`, `contracts/facade/freenet.toml`, `contracts/facade/rust-toolchain.toml`, `contracts/facade-types/src/lib.rs`, `contracts/facade-types/Cargo.toml`, `contracts/facade-loader/README.md`, `contracts/facade-loader/src/index.html.tmpl`, `common/Cargo.toml`, `common/src/lib.rs`, `common/src/facade.rs`, `common/src/web_container.rs`, `contracts/.cargo/config.toml`, `modules/.cargo/config.toml`, `published-contract/README.md`, `legacy_inbox.toml`, `legacy_identity_delegate.toml`, `legacy_local_state_delegate.toml`

Skimmed: `contracts/inbox/tests/integration.rs (module docs, all 40 test names and doc comments, bodies at lines 1-320, 893-980, 1556-1606, plus a grep of every entry-point call)`, `modules/mail-local-state/src/lib.rs (non-test code, lines 1-885, read in full; tests 886-1359 by name and doc comment, first eight bodies)`, `modules/mail-local-state/src/bin/tool.rs (diffed against the identity tool: differs only in names and help text)`, `modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp (outline only; not compiled)`, `Outside owned paths, ranges only: ui/src/aft.rs:184-204, 466-512; ui/src/api.rs:580-596, 1296-1325; ui/src/inbox.rs:71-83; ui/src/local_state.rs:722-742; ui/src/migration/legacy_delegates.rs:76-92; ui/build.rs:45-80; Makefile.toml:28-50, 900-985; scripts/check-excluded-crates.sh; scripts/check-wasm-builds.sh:1-60; scripts/build-loader.sh:1-40; wire-v1/src/lib.rs:1-40; wire-v1/capture/Cargo.toml and src/main.rs:1-40; root Cargo.toml:1-50; freenet-stdlib 0.3.5 registry source (prelude re-exports, set_secret, OutboundDelegateMsg)`, `Sibling maps consulted by grep: ui-core-v1.md (first 60 lines), ui-migration.md, build-ci-release.md, delegate-core.md`

Skipped: `All Cargo.lock files under the owned paths (told to skip)`, `target/ directories under the owned crates (told not to touch)`, `modules/antiflood-tokens/interfaces/examples/rsa4096-pub.pem (leftover RSA public key, data not source)`, `published-contract binaries and hash files other than README.md (WASM; code-hash files were only printed)`
