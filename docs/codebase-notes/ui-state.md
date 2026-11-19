> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Map: ui-state (mail delegate client, local mail state, identity/contact stores, jobs loop)

Paths are relative to the repo root. All line numbers are at upstream 2358b59.

## 1. Purpose and where this sits

The page (Dioxus, WASM) holds no private keys and no durable state. Everything durable goes through one request/response protocol (`freenet-mail-protocol`, CBOR) to the mail delegate. This subsystem is the page's side of that:

- `ui/src/mail_delegate/` is the client: one `call(Op) -> Reply`, two transports (node, in-page), typed wrappers.
- `ui/src/local_state.rs` + `local_state/items.rs` are the page's in-memory mirror of drafts/sent/kept/archived/flags/settings and the writers that persist each change as one delegate item.
- `ui/src/app/identity_store.rs` loads identities, plans v1 contract migrations from markers, and drives rename/delete/repair/publish.
- `ui/src/app/contact_store.rs` persists contacts as items; `ui/src/contact_tier_cache.rs` is an unrelated in-memory cache of recipients' v1 inbox policy.
- `ui/src/jobs.rs` asks the delegate to run its background jobs while a page is open.

The node loop (`ui/src/api.rs`) is what delivers delegate answers, so nothing here may be awaited inside it (ops.rs:6-8, identity_store.rs:5-8); results return as `NodeAction`s (`send_action`, identity_store.rs:30-35).

## 2. Walkthrough

### ui/src/mail_delegate/mod.rs (client core)
- `CallError::{Failed(Failure), Unknown(String)}` (32-38). `Unknown` = no answer; never "absent". `Display` prints `"{code:?}: {detail}"` (43); `cap_reached_text` string-matches `"CapReached:"` in that text (57-61), pinned by contact_store.rs:653-670.
- `new_request` (115-121): `Request{version: PROTOCOL_VERSION(=1), request_id: 16 random bytes from getrandom (rng.rs:9-13), op}`.
- `call` (176-203): encode CBOR, insert a oneshot waiter in thread-local `PENDING` keyed by request id (96, 191), hold a `PendingSlot` whose `Drop` removes it (166-173), `send`, then `wait`.
- `timeout_ms` (81-90): 30 000 ms by default; `(600+30)*1000` for the five prompting ops (ExportRecoveryKey, ExportProfile, ExportForSuccessor, DeleteIdentity, RepairIndex). `DELEGATE_PROMPT_TTL_SECS = 600` (75) mirrors `store::PROMPT_TTL_SECS` (delegates/mail-delegate/src/store.rs:56). The same five ops are `consented` (103-112) and tracked in `CONSENTED`.
- `wait` (293-316): on wasm, races the receiver against a gloo timer; timeout text is "the node is busy and did not answer in time; try again in a minute" (308). On native there is NO timeout (311-315).
- `deliver` (126-147): decodes only `ResponseId{request_id}` first (159-162) so a late reply (e.g. a recovery key) nobody waits for is never decoded into a `Reply`.
- `fail_pending_calls` (242-251): drops every non-consented waiter, ending those calls as `Unknown("the node reported an error running the mail delegate")`. `may_be_mail_delegate_error` (211-222) decides whether a node `DelegateError` (which carries no request id) may be the mail delegate's.
- `reply_name` (266-291) is an exhaustive match over `Reply`: a new variant fails to compile here.
- `send` picks the backend by cfg (318-332): `use-node` → `node::send`; else `local-delegate` → `local::process`; neither → `Err("this build has no mail delegate")`.
- Test counters `V1_AFT_PARAMS_CALLS`, `RUN_JOBS_CALLS` (149-156).

### ui/src/mail_delegate/node.rs (node transport)
- `CODE`/`CODE_HASH` are `include_bytes!`/`include_str!` of `published-contract/candidate/mail_delegate{,_code_hash}` (19-22). Parameters are empty (30-32); the key is `DelegateKey::from_params(CODE_HASH, [])` (34-40).
- Registration once per page load: `Registration::{NotStarted, Waiting{ack, waiters}, Done}` (42-50), `ensure_registered` (116-162). Constants: `REGISTER_ACK_TIMEOUT_MS = 10_000` (26), `REGISTER_WAIT_TIMEOUT_MS = 25_000` (111). `Attempt`'s `Drop` settles a cancelled attempt as failed (90-107).
- `register_request` sends `RegisterDelegate{delegate, cipher: V0_2_CIPHER, nonce: V0_2_NONCE}` (201-211; constants at api.rs:570-578).
- `send` (225-242): `DelegateRequest::ApplicationMessages{key, params, inbound: [ApplicationMessage(bytes)]}` over `WEB_API_SENDER`.
- Inbound: api.rs:3424-3442 routes `HostResponse::DelegateResponse` for the mail key: each `ApplicationMessage` payload → `deliver`; a response with none → `note_registration_answer`. Errors: api.rs:2672-2696.

### ui/src/mail_delegate/local.rs (offline transport)
- The real delegate crate (`mail_delegate::handlers::process`) run in the page over `PageEnv` (in-memory `BTreeMap` secrets + contract states, 22-32, 62-99). Fixed origin `ORIGIN = [0x4d; 32]` (20). Nothing persists across reload.
- Consent prompts are auto-answered with the first choice (144-154).
- `set_contract_state` (107-109) lets `ui/src/v2/net.rs` expose offline inboxes to the delegate. Test hooks: `fail_writes_containing`, `miss_next_reads`, `forget_contract_state` (37-39, 58-60, 114-116).

### ui/src/mail_delegate/ops.rs (typed wrappers)
One function per op, each accepting only that op's reply kinds:
- Identities: `list_identities` (20), `get_status` (29), `create_identity` → `(IdentityInfo, SecretString)` (38-54), `after_lost_create`/`find_lost_create` (85-120; re-check after `LOST_CREATE_RECHECK_MS = 3_000`, 57), `import_identity` → `(info, already_present)` (124-143), `update_identity`/`confirm_identity` → new generation (145-163), `delete_identity` loops on `Reply::Continue{remaining}` up to 1 000 times (169-186), `list_deleting_identities` (190).
- Items: `get_item` (221), `list_all_items` pages with `limit: 256` (230-254), `put_item` (258-271; `None` = create-only, `Some(g)` = CAS, `Some(0)` = must be absent), `delete_item` (273-279), `put_item_merged` (293-336): read-merge-write with `MERGE_RETRIES = 3` (282) and pauses 100/200/400 ms; an item that existed at first read and is gone on a retry is NOT recreated (`Ok(None)`); `Unreadable` → merge onto `None` and write with `expected_gen: None`, plus a once-per-page toast (`notice_unreadable`, 208-219). All item writes hold a `pending_saves` guard (264, 274, 297).
- v1 crypto: `decrypt_v1` batches of `MAX_DECRYPT_BATCH = 32`, re-asks entries answered `CapReached` (351-401); `seal_v1` (412-435); `sign_v1_authority` (439-450); `sign_v1_removal` sorts+dedups hashes first (454-472); `v1_aft_params` (538-543) returns the token-generator parameters, which carry the ML-DSA seed.
- Prompting: `export_recovery_key` (547), `export_profile` (556, dead code), `repair_index` (567), `revoke_export_grant` (578).
- `run_jobs` (599-604). `SecretBytes`/`SecretString` zeroize and redact `Debug` (485-526).
- `pause_ms`/`now_ms` (613-626): on native a pause does not sleep, it advances a thread-local clock.

### ui/src/local_state.rs (in-memory model)
- `SNAPSHOT: LocalState` (34) is what views read; `GENERATION` signal (349) is bumped on every change (351-359).
- State is keyed by `addr_key` = `"@" + 48 hex` of the identity address (231-239), never by alias; `state_key(alias)` resolves through the live identity list, then the inbox routing map, else returns the alias itself (258-267).
- Every mutation is a `Change` (93-110) applied by `apply` (114-186) through `change()` (208-218), which also journals it while a load runs (`JOURNAL`, `LOADS`, 84-88).
- Load protocol: `begin_load() -> LoadMark` (289-292), `merge_loaded(loaded, owners, mark)` (303-342): replaces only the owners it read, replays every journaled change since the mark, then `install_snapshot(merged, false)`.
- Optimistic stashes re-applied by `install_snapshot` (371-461): `DELETED_DRAFTS` (47), `PENDING_KEPT` (56), `OPTIMISTIC_SENT` (65), `OPTIMISTIC_ARCHIVED` (72), `DELETED_MESSAGES` (80). Sent/kept/archived stashes are dropped when the delegate acks the write (`drain_acked_*`, 940-958).
- `local_*` functions (758-989) are the synchronous optimistic half of each write. `local_set_sent_delivery_state` returns whether the row transitioned; `Delivered`/`Failed` are terminal (811-833, 874-876).
- `UNREAD_IMPORTS` (39-40) / `is_unread_import` (480-483): ids whose stored flags are `read=false, deleted=false`. Filled only by `items::load` (items.rs:893-910). The inbox merge shows a kept copy as read unless `is_unread_import` (app.rs:1359); `is_read` overrides it.
- Read marks: `should_persist_read` once per page via `READ_SENT` (572-585); `persist_read` (641-653, `use-node` only) spawns a forever-task with `READ_MARK_ATTEMPTS = 3` (589) for transient failures (`Unknown`, `Busy`, `Conflict`; 596-606), pauses 1 s, 2 s.
- `stranded_pending_sent` (847-869): Pending rows older than the caller's timeout with no send of this page waiting.
- `persist_identity_settings`/`persist_global_settings` (1003-1034): optimistic patch, then (`use-node` only) a field-level merged write.
- `Stored::{Yes, Failed, Unknown}` (709-726) classifies a write result for the "Deleted/Archived" confirmations (app.rs:5470-5516).
- `DRAFT_RECIPIENTS` (546-568): the recipient address recorded per draft.

### ui/src/local_state/items.rs (item layout and writers)
- Ids (10-21): drafts and sent = the 16 UUID bytes; received messages (kept, archived, flags) = `blake3(ciphertext)[..16]` (90-94). The UI `MessageId` is the first 8 bytes LE with the top bit cleared (71-76; `RECEIVED_ID_LIMIT = 1<<63`, local_state.rs:224); the full 16 bytes are remembered per page in `FULL_IDS` (42) and a write for an id not in it fails "message N is not loaded" (96-100).
- Bodies are JSON: `StoredDraft` = wire-v1 `Draft` flattened + optional `to_address` (243-249); `StoredSent` = `SentMessage` flattened + `presumed_failed` omitted when false (426-441); `KeptMessage`, `ArchivedMessage` bare; `Flags{read, deleted}` (122-126); settings records with only changed leaf fields merged (`apply_changes` 617-648, `merge_changed_fields` 653-664).
- `save_draft` (300-344) → `DraftSave::{Stored, Discarded, GoneElsewhere}`; `SEEN_DRAFTS` (217) lets a page know a draft it saw stored was removed elsewhere; "gone" needs two reads to agree. `check_draft_before_send` (266-283).
- `save_sent` (390-411) with `EARLY_DELIVERY` (372) for a delivery state that beat the item; `advance_sent` (446-456): terminal states stick, except a presumed `Failed` yields to `Delivered`. `presume_sent_failed` (473).
- `mark_read` (501-538): flags first, then the kept copy, then a re-check that takes the copy back if a delete raced it. `archive_message` (541-561): archived copy, delete kept, flags read. `delete_message_after_copies` (588-612): kept, archived, `between()`, flags `deleted=true`, then kept again.
- `delete` (156-191): absent only when two reads agree; unreadable → `delete_item(item, 0)`.
- `reclaim_flags` (716-751): drops flags with no kept/archived copy, not live in any inbox state, not written since the reclaim began (`FLAG_WRITES`, 45).
- `load` (858-925): per identity lists Draft, Sent, Kept, Archived, Flags, reads IdentitySettings; then GlobalSettings. A list answering `Busy` after 2 retries (400, 800 ms) is recorded in `UNREADABLE_LISTS` and loads empty (819-848).

### ui/src/app/identity_store.rs
- `load_and_report` (783-799): `load_identities`, then (use-node) spawns `jobs::fallback()` and awaits the migration walk; `list_then_walk` fixes the order (803-809).
- `load_identities` (813-900): list with `STARTUP_LOAD_RETRIES = 3` (225; pauses 1/2/4 s); status check for unreadable identities; `v2::owner::start` per confirmed identity (849-851); `prepare` each; then `announce_then_merge` (921-942): `begin_load` → `IdentitiesLoaded` → unfinished-deletion check → `local_state::load` → `merge_loaded`; then `inbox::fail_stranded_pending_sent`. Offline builds only announce (895-899).
- `MigrationMarkers` (544-564) live in the `IdentitySettings` item body under `"migration"` (603-630). `plan_migration` (676-695), `plan_load` (949-968), `prepare` (973-1004) → `LoadedIdentity{info, inbox_migrate_from, aft_migrate_from}` (661-665), consumed at api.rs:1924-1953. `migration_landed` clears a pending marker (1007-1040).
- Page view of an identity: `IdentityInfo{addr, vk, ek, alias, description, confirmed, generation}` (protocol/src/lib.rs:539-550) → `Identity::from_info` (ui/src/app/login.rs:88-101).
- Rename (69-107), paged delete with progress (128-158), unfinished deletions (190-218, 401-465), index repair (286-327), `publish`/`publish_or_recover` (1072-1150), `initial_inbox_state` (1044-1065).

### ui/src/app/contact_store.rs
- One `ItemKind::Contact` item per contact, `owner: None`, id = `address_of(vk)` (59-65). Body: JSON `ContactRecord{v: 1, local_alias, description, keys: StoredContactKeys, v2?: V2Notes}` (19-40); `decode` refuses a record stored under another address (97-110).
- `save` uses `store_replacing` (199-236), deliberately not `put_item_merged`: a contact deleted mid-save is recreated. `remove` (239-265).
- Per-contact FIFO `KeyQueue` (340-374) + `enqueue` (397-434) keep background writes in user order. `CHANGED_DURING_LOAD` journal (117-158). `disambiguate` relabels clashing aliases as `alias~<6 hex>` (285-329).

### ui/src/contact_tier_cache.rs
Process-global `RwLock<HashMap<ContractKey, CacheEntry>>` (62-63) of each contact's v1 `InboxSettings` (tier, max age, verified-sender bypass), last-write-wins by `last_update` (73-111). Fed from api.rs:2808, 3158, 3191; read by inbox.rs:787 and app.rs:6325. No delegate involvement.

### ui/src/jobs.rs
`fallback` (102-133): once per page (`claim_start`, 96-98), `RunJobs` then pause `PERIOD_MS = 600_000` (25), forever; failures are logged only. With `e2e-probe` it also probes `get_status` 100 ms in and publishes `data-jobs-run` JSON (136-175). `PROBE_MARKER` (35-39).

## 3. Interfaces

- Wire: CBOR `Request`/`Response` (protocol/src/lib.rs:104-115), `Op` (228-435), `Reply` (575-633), `FailureCode` (124-144), `ItemKind` (149-187). Limits: `MAX_RESPONSE_BYTES` 512 KiB, `MAX_ITEM_BYTES` 256 KiB (28-30).
- Item bodies are JSON of `freenet-mail-wire-v1` types (wire-v1/src/local_state.rs:42-143, 256-277, 453-460) plus the UI-side flattened extras above.
- Implicit contract with the delegate's v1 watch: flags JSON shape and message id derivation are duplicated in `delegates/mail-delegate/src/watch.rs:301-314` and items.rs:90-94, 122-126; the watch writes `{"read":false,"deleted":false}` then a create-only kept copy (watch.rs:385-403).
- Delegate storage keys (for tests that inject faults): `o/<origin hex>/it/<tag>/<owner hex|->/<id hex|->`, tags `ct dr se kp ar fl si sg …` (store.rs:103-105, 145-163, 266-277). Caps: 2048 contacts, 256 drafts, 4096 each of sent/kept/archived/flags per identity, 16 identities (store.rs:36-42).
- stdlib: `DelegateRequest::{RegisterDelegate, ApplicationMessages}`, `InboundDelegateMsg`, `OutboundDelegateMsg`, `DelegateError`, `DelegateKey`.
- Features (ui/Cargo.toml:99-122): `use-node` (default), `local-delegate` (pulled by `example-data`, `no-sync`), `e2e-probe`, `test-ghost-master`. `local-delegate` + `use-node` is a `compile_error!` (ui/src/lib.rs:16-20).
- Callers in: api.rs (deliver/errors/contacts load), app.rs (all local_state writes), app/login.rs, app/settings.rs, app/migration_view.rs, migration/ (`PageStore` calls `mail_delegate::call` directly, migration/mod.rs:78-84), v2/{contacts,owner,send,queue}.rs (v2 ops have no ops.rs wrappers; they use `call` + `unexpected`). `v2::owner::publish` also sends `RunJobs` after each inbox PUT (v2/owner.rs:414-416).
- DOM attributes read by Playwright: `data-pending-saves` (pending_saves.rs:60-62), `data-jobs-run` (jobs.rs:173).

## 4. Invariants and where enforced

| Rule | Code | Pinned by |
|---|---|---|
| A reply reaches only the call with its id | mod.rs:126-147 | mod.rs:421-439 |
| No answer is Unknown, never absent/failed | mod.rs:196-202; `Stored::of` local_state.rs:717-725 | identity_store.rs:1636-1653, 1730-1746 |
| A consented call outwaits the delegate prompt and survives `fail_pending_calls` | mod.rs:81-90, 242-251 | mod.rs:347-390 |
| Loading an identity never fetches the signing seed | identity_store.rs:973-1004 | ops.rs:806-817 |
| A merged write never resurrects a deleted item | ops.rs:315-319 | ops.rs:736-767 |
| A late autosave never restores a discarded draft; a draft removed elsewhere is not recreated or re-sent | items.rs:300-344, 266-283 | items.rs:1513-1668 |
| Deletion removes copies before setting the mark | items.rs:588-612 | items.rs:1673-1727 |
| Sent delivery states are terminal except presumed Failed | items.rs:446-456; local_state.rs:811-833 | items.rs:1115-1130, 1182-1229; local_state.rs:1985-2055 |
| A load replaces only what it read and replays page changes | local_state.rs:303-342; identity_store.rs:921-942 | local_state.rs:1305-1509; identity_store.rs:1567-1605 |
| Unreadable markers plan nothing and write nothing; held markers are not acted on | identity_store.rs:949-968 | identity_store.rs:1830-1833; guard "held markers are not acted on" (scripts/check-guards-fail.sh:1193) |
| Listing finishes before the migration walk is polled | identity_store.rs:803-809 | migration/tests.rs:2508; guard at check-guards-fail.sh:1318 |
| A finished deletion is recorded for scrub | identity_store.rs:151-153 | identity_store.rs:1157-1175; guard :1516 |
| Unread only from flags that say so | items.rs:893-910 | items.rs:1792-1822 |
| Contact record must match its address; contact writes run in order | contact_store.rs:97-101, 340-434 | contact_store.rs:531-553, 735-740 |
| Register with the v0.2 cipher/nonce | node.rs:201-211 | node.rs:290-299 |
| Embedded WASM matches embedded code hash | node.rs:19-22 | node.rs:248-257 |
| Probe marker only under `e2e-probe` | jobs.rs:35-39, 106-107 | jobs.rs:181-191; three guards at check-guards-fail.sh:1707-1719; scripts/check-release-scrub_test.sh:57-58 greps the source text |

## 5. State: shipped, candidate, unfinished

- Nothing in this subsystem is shipped: the UI talks to a candidate delegate (node.rs:17-22) and releases are blocked by `.release-freeze`. Workspace version is still 0.2.7 (Cargo.toml).
- Frozen inputs this code depends on: wire-v1 shapes (`wire-v1/src/golden_v1.txt`), the v0.2 register cipher/nonce (`ui/src/register_defaults_v1.txt`), the legacy local-state delegate key pin (local_state.rs:731-753).
- Append-only: `protocol/src/golden_v1.txt` (95 lines).
- Transitional, must go before v0.3.0: `ops::v1_aft_params` and its use in `identity_store::publish` (1122-1126). `scripts/check-release-delegate.sh` refuses a frozen delegate built with `v1-transition`, and without that feature the delegate answers `Refused` (handlers.rs:1597-1600), which would fail every identity publish.
- Unused: `export_profile` (ops.rs:554-561), `unarchive_message` (items.rs:563-568; #60), `Op::ExportForSuccessor` (no UI caller).

## 6. Tests

All tests are inline unit tests; two feature sets are needed because modules are cfg-gated.

- Default (`use-node`): `cargo test -p freenet-email-ui --lib` (part of `cargo make test-workspace`). Runs mod.rs (9 of 10), node.rs (5), local_state.rs (38, including the 2 legacy key pins), items.rs `tests` (10), most of identity_store.rs, contact_store.rs (12), contact_tier_cache.rs (5), jobs.rs (3). Any real `call()` fails at once ("not connected to the node").
- Offline: `cargo make test-ui-offline` = `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` (Makefile.toml:916-919). Adds everything that drives the real delegate in-page: ops.rs `tests` (15: lost create, pending-save counting, CAS, decrypt/seal interop with the v0.2 client, AFT params byte-equality), local.rs (4: paged deletion, interrupted deletion), items.rs `delegate_tests` (21), 3 identity_store tests, 2 contact_store tests, the prompt-TTL pin.
- `cargo make test` runs both plus excluded crates. Prerequisites: toolchain 1.99.1 (rust-toolchain.toml); no node. `mail-delegate` is a dev-dependency in every feature set (ui/Cargo.toml:124-127).
- Fault injection is by key substring: `local::miss_next_reads("it/dr/…", 2)` etc.
- Real-node: `cargo make test-e2e-background` (builds with `e2e-probe`, reads `data-jobs-run`; CI `e2e-background.yml` triggers on `ui/src/jobs.rs`, `ui/src/mail_delegate/**`, `ui/src/local_state/**`), `cargo make test-e2e-real-node`. Offline Playwright (`cargo make test-ui-playwright`): `ui/tests/mail-delegate-identity.spec.ts`, `export-permission.spec.ts`.
- Not covered by any unit test: the wasm-only timers (mod.rs:297-310; node.rs:166-175, 185-194) and a `call()` dropped mid-wait (admitted at mod.rs:454-458).

## 7. Changing it safely

- New or changed `Op`/`Reply`/`ItemKind`: edit `protocol/`, append to `protocol/src/golden_v1.txt`, extend `reply_name` (mod.rs:266-291) and `list_name` (items.rs:777-793, exhaustive), decide `timeout_ms`/`consented`, then rebuild the delegate candidate (`published-contract/candidate/README.md`); `scripts/check-delegate-candidate.sh` fails otherwise. The rebuild changes `CODE_HASH`, hence the delegate key: data a dev node stored under the previous candidate is not reachable from the new build.
- Because the key is the hash of the embedded WASM, the page always talks to exactly the delegate it embeds; there is no runtime version skew on the node path. After the v0.3.0 freeze any protocol change is a new delegate generation plus a data migration.
- Never change a wire-v1 struct to add a stored field. Use the existing pattern: a UI-side wrapper with `#[serde(flatten)]` and an omitted-when-default field (items.rs:243-249, 426-432), or an extra JSON key merged with `put_item_merged` (identity_store.rs:603-654).
- The flags JSON and the message-id derivation are duplicated in the delegate (watch.rs:301-314). Changing either side alone silently breaks imports, unread state and delete tombstones.
- Any new item write must go through `ops::put_item`/`put_item_merged`/`delete_item` so `pending_saves` counts it, and must run in `spawn_forever`/`spawn_detached`, never a scoped `spawn` and never in the node loop.
- Any new page-side per-identity cache needs clearing in `forget_key` (local_state.rs:527-538) and `forget_everywhere` (identity_store.rs:371-379).
- Guards mutate exact source text: identity_store.rs:152, 807-808, 952 and jobs.rs:35, 106-107. Reformatting those lines breaks `check-guards-fail.sh`; `check-release-scrub_test.sh` greps jobs.rs too.
- Name trap: inside `ui`, `mail_delegate::…` is the delegate crate, `crate::mail_delegate::…` is this client (e.g. mod.rs:352, items.rs:1878).
- Most v1 local-state persistence call sites are `#[cfg(feature = "use-node")]` (e.g. app.rs:6038, local_state.rs:1007-1016, identity_store.rs:842-894); the offline UI keeps that state in `SNAPSHOT` only. Verify a change in both feature sets.
- Page state is thread-local; tests isolate with unique aliases and `fresh_snapshot()`. `contact_tier_cache::CACHE` is process-global (unique keys per test, contact_tier_cache.rs:197-201).

## 8. Docs/comments vs code

1. local_state.rs:4-12 says the snapshot is "loaded once per page" and keyed "by alias string". It is reloaded on every `load_identities` (identity_store.rs:813-875; called again from migration_view.rs:139-190, confirm at :749, repairs at :294, :317) and keyed by `@<addr hex>` (local_state.rs:226-239).
2. local_state.rs:42-80 describe stashes dropped "when the delegate echoes". `replace_snapshot` is `#[cfg(test)]` (363-366); production installs only with `confirm_discards = false` (341), so draft tombstones are never dropped, and the dozen "stale echo" tests exercise a test-only entry point. Partly admitted at 28-33.
3. node.rs:113-115 "A failed attempt is retried by the next request": only a failed send is. An attempt that was sent ends `Done` whether or not the node acknowledged or refused it (151-158).
4. protocol/src/lib.rs:152 says Draft ids are "a content hash"; drafts and sent use UUID bytes (items.rs:10-11, 109-111).
5. ops.rs:174-175 "about 17k items" per identity predates the v2 kinds (store.rs:224-240).
6. scripts/check-release-delegate.sh:15 says releases before 0.3.0 "do not ship this delegate", but the default build embeds the candidate now (node.rs:19-22). Only `.release-freeze` prevents a 0.2.x release shipping it.
7. AGENTS.md cites prime GETs at api.rs "~1442 and ~2540"; they are at api.rs:420-430 and :2277.
8. AGENTS.md's Consent bullet omits `RepairIndex`, which also prompts (protocol/src/lib.rs:356-361; mod.rs:87).
9. local.rs:1-5 says the offline build exercises the same protocol; for v1 local mail state the offline UI does not call the item writers (see section 7), only unit tests do.
10. `.release-freeze` says the background jobs and inbox v2 "are still to land"; both are in the tree (ui/src/jobs.rs, ui/src/v2/).
11. app/login.rs:64-66 says the AFT tier is hashed into `InboxParams`; `InboxParams` holds only `pub_key` (wire-v1/src/inbox.rs:37-39).
12. Misplaced comments: local_state.rs:693 (describes `presume_sent_failed`, sits on `mark_read`), test docs at 1106-1111 and 1303-1305.

## 9. Suspected weak spots

- (medium) A registration the node refuses is never retried in the page: `REGISTRATION` goes `Done` after the 10 s ack wait regardless (node.rs:151-158) and the error path (api.rs:2672-2696) does not reset it. Every later call then ends `Unknown` until reload. Depends on how core reports it.
- (medium) After a reload `Identity::from_info` hard-codes `required_tier: Min10` and the default max age (login.rs:93-94), and the #288 self-heal re-PUT builds the inbox from those (api.rs:3668-3672 → identity_store.rs:1044-1065), so a self-healed inbox may lose the user's chosen policy.
- (low-medium) Later loads replay only changes journaled since that load began. Drafts and settings have no optimistic stash, so a draft/settings write still in flight when a migration- or repair-triggered `load_identities` starts can be reverted in `SNAPSHOT` until the next edit or reload (local_state.rs:303-342). The stored copy is unaffected.
- (low) `items::load` writes `DRAFT_RECIPIENTS`, `SEEN_DRAFTS` and `UNREAD_IMPORTS` directly, outside the journal (items.rs:868-869, 910). A recipient chosen during the load can be overwritten by the stored one; sending then refuses rather than misdelivers (app.rs:6241-6248).
- (low) `state_key` treats any alias of the form `@` + 48 hex as a key (local_state.rs:259-261); the delegate allows such an alias (handlers.rs:356-361) and the create form only trims (login.rs:1532-1536).
- (low) A keyless delegate error while a Ghost Key call is in flight ends all mail calls as Unknown: api.rs:2689 passes only `token_call`.
- (low) `DELETED_MESSAGES` is never drained on ack (no counterpart to `drain_acked_*` in items.rs:580-612) and `forget_key` does not clear `UNREAD_IMPORTS`; `FULL_IDS` only grows.
- (low) `mark_read` returns `Err` after the flag was stored when the kept copy is refused (items.rs:511-531); the user is told the read mark may not persist, and the `PENDING_KEPT` body is held for the session.
- (low) `contact_store::decode` sets v2 notes as a side effect (102-104) before `unchanged_since_load_began` filters; a v2 contact removed during the startup load can leave stale notes.
- (adjacent) api.rs:2697-2699 `RequestError::Disconnect => todo!()` panics the node loop that delivers every answer here.

## 10. Open work and documented limits

- v1 AFT path removal (ops.rs:528-537; identity_store.rs:1122-1138) before the delegate can be frozen without `v1-transition`.
- Candidate → `published-contract/wasm/v2/` and repointing node.rs:19-22 (node.rs:17-18).
- JSON profile export has no UI (ops.rs:554-555); true unarchive is #60 (local_state.rs:701-703).
- A per-create nonce would close the lost-create alias ambiguity "at the cost of a delegate change" (ops.rs:77-84).
- Delivery-state gaps under #58 (wire-v1/src/local_state.rs:25-27); kept copies do not persist threading metadata (app.rs:1303-1305).
- contact_tier_cache is in-memory only and re-primed per load (contact_tier_cache.rs:14-15).
- A timed-out call is still queued in the node and may run later; the page reports it as busy (mod.rs:304-308).
- A background import that lands while the page is open is known as "unread import" only after the next load.

## 11. Questions the tree does not answer

1. How freenet-core answers a failed `RegisterDelegate` and an `ApplicationMessages` for an unregistered delegate (keyed error or not), which decides whether the registration weak spot can stick.
2. Whether the self-heal policy reset is intended, or the tier is restored somewhere not found.
3. The intended order for v0.3.0: remove the UI's `V1AftParams` use first, or freeze a delegate that still has `v1-transition`.
4. Where the "8 queued calls" figure in AGENTS.md comes from (not in this tree; presumably core).
5. Whether repeated `load_identities` runs are meant to be safe against in-flight draft/settings writes.

---

## Appendix: the reader's structured summary

This is the page's whole durable-state layer. Line references below are relative to /home/user/mail-freenet.

`ui/src/mail_delegate` is a CBOR request/response client for the mail delegate: random 16-byte request ids, a waiter map, and 30 s default / 630 s consent timeouts that exist on wasm only. It has two transports: a node (registers and messages the candidate WASM embedded from `published-contract/candidate`), and for offline builds the real delegate crate run in-page over an in-memory store. `ops.rs` wraps each v1 and identity op; the inbox v2 ops call `call` directly.

`local_state.rs` keeps an optimistic in-memory `LocalState` keyed by `@<address hex>`, journals changes made during a load and replays them over it. `items.rs` persists each change as one JSON item (wire-v1 shapes plus UI-side flattened extras) through compare-and-swap merged writes. It also defines message ids and the flags shape, which it shares implicitly with the delegate's v1 watch.

`identity_store.rs` loads identities, plans v1 contract migrations from markers kept inside the IdentitySettings item, and drives rename, paged deletion, index repair and publish. `contact_store.rs` stores one item per contact with ordered background writes. `jobs.rs` sends `RunJobs` on load and every ten minutes.

Nothing here is released: the delegate is a candidate, releases are frozen by `.release-freeze`, and the transitional `V1AftParams` call that identity publish needs conflicts with the v0.3.0 release guard. Tests are inline unit tests split across two feature sets (default and offline); the wasm-only timeout paths have no unit coverage. I ran nothing, per the read-only rule.

### Key facts

- Every delegate call is `call(Op)`: a CBOR `Request{version:1, request_id: 16 random bytes, op}`, a oneshot waiter in thread-local PENDING keyed by id, and a two-step decode of the reply (id first, so a late reply is never decoded). ui/src/mail_delegate/mod.rs:115-147, 176-203
- Timeouts are 30 s by default and 630 s for the five prompting ops (ExportRecoveryKey, ExportProfile, ExportForSuccessor, DeleteIdentity, RepairIndex). They exist only on wasm; native builds (all unit tests) never time out. The 'node is busy' text is the timeout message. ui/src/mail_delegate/mod.rs:81-90, 293-316 (text at :308)
- A node DelegateError carries no request id, so `fail_pending_calls` ends every non-consented waiting call as CallError::Unknown; attribution is `may_be_mail_delegate_error`. ui/src/mail_delegate/mod.rs:211-251; ui/src/api.rs:2672-2696
- The node transport embeds the candidate WASM and code hash, uses empty parameters (key = hash of the WASM), and registers once per page load with the v0.2 cipher/nonce; ack wait 10 s, waiter timeout 25 s. Answers are routed in api.rs. ui/src/mail_delegate/node.rs:19-40, 26, 111, 116-162, 201-242; ui/src/api.rs:3424-3442
- The offline transport runs the real delegate crate in the page over an in-memory store with fixed origin [0x4d;32], auto-answers consent prompts with the first choice, and persists nothing. `local-delegate` with `use-node` is a compile_error. ui/src/mail_delegate/local.rs:20, 123-158; ui/src/lib.rs:16-20
- Item write semantics: put_item with None is create-only, Some(g) is compare-and-swap, Some(0) means must be absent. put_item_merged retries 3 conflicts (100/200/400 ms), never recreates an item deleted meanwhile, and replaces an Unreadable one. Every item write holds a pending_saves guard. ui/src/mail_delegate/ops.rs:258-336; delegates/mail-delegate/src/handlers.rs:1152-1196
- The local model is a thread-local SNAPSHOT plus a GENERATION signal, keyed by `@`+48 hex of the identity address, never by alias. Changes made during a load are journaled and replayed by merge_loaded; five optimistic stashes are re-applied on install. ui/src/local_state.rs:34, 84-88, 231-267, 289-342, 349, 371-461
- Item ids: drafts and sent use the 16 UUID bytes; kept/archived/flags use blake3(ciphertext)[..16]. The UI MessageId is the first 8 bytes LE with the top bit cleared, and a write needs the full id remembered in FULL_IDS this page. Bodies are JSON of wire-v1 types with UI-side flattened extras (to_address, presumed_failed). ui/src/local_state/items.rs:10-21, 71-100, 122-126, 243-249, 426-441
- There is an implicit contract with the delegate's v1 watch: the flags JSON shape and message-id derivation are duplicated in delegates/mail-delegate/src/watch.rs:301-314, and an import writes unread flags then a create-only kept copy (watch.rs:385-403). `is_unread_import` comes only from flags read=false and deleted=false seen at load. ui/src/local_state/items.rs:893-910; ui/src/local_state.rs:480-483; ui/src/app.rs:1359
- Sent delivery states: Delivered and Failed are terminal, except a presumed Failed (a stranded row older than 90 s) yields to Delivered; a state that beats its Sent item waits in EARLY_DELIVERY. ui/src/local_state/items.rs:372-499; ui/src/local_state.rs:811-876; ui/src/inbox.rs:254, 323-348
- Write orderings that matter: delete removes kept and archived copies, then sets the deleted flag, then deletes kept again; mark_read writes flags, then the kept copy, then re-checks; `delete` treats an item as absent only when two reads agree. ui/src/local_state/items.rs:156-191, 501-538, 588-612
- Startup path: load_and_report runs load_identities (3 retries), prepares each identity, then announce_then_merge (begin_load, IdentitiesLoaded, unfinished-deletion check, local_state::load, merge_loaded), then fails stranded sends. load_identities is re-run by the migration walk, confirm and repairs. ui/src/app/identity_store.rs:783-900, 921-942; ui/src/app/migration_view.rs:138-167
- MigrationMarkers live inside the IdentitySettings item JSON under "migration". plan_load plans nothing for unreadable or held markers; prepare writes markers before the node loop dispatches migration GETs, which happens at ui/src/api.rs:1924-1953. ui/src/app/identity_store.rs:544-654, 676-695, 949-1004
- An identity reaches the page as IdentityInfo{addr, vk, ek, alias, description, confirmed, generation} with no key material. Identity::from_info hard-codes required_tier Min10 and the default max age; generation is the compare-and-swap token for rename/confirm/delete and upsert keeps the newer generation. protocol/src/lib.rs:539-550; ui/src/app/login.rs:60-101, 160-199
- Contacts are one ItemKind::Contact item each (owner None, id = address_of(vk)), JSON ContactRecord v=1 with optional v2 notes; saves recreate a contact deleted mid-write and run through a per-contact FIFO. contact_tier_cache is a separate process-global in-memory cache of recipients' v1 inbox policy. ui/src/app/contact_store.rs:19-40, 59-65, 182-236, 340-434; ui/src/contact_tier_cache.rs:62-111
- The jobs loop starts once per page, sends RunJobs, then pauses 10 minutes, forever; failures are only logged. With e2e-probe it publishes data-jobs-run. The page also sends RunJobs after each v2 inbox PUT. ui/src/jobs.rs:25, 96-133, 159-175; ui/src/v2/owner.rs:414-416
- identity_store::publish calls ops::v1_aft_params for every new or restored identity, but a delegate built without `v1-transition` answers Refused and the release guard requires a frozen delegate without that feature. The v1 AFT path must leave the UI before v0.3.0. ui/src/app/identity_store.rs:1122-1126; delegates/mail-delegate/src/handlers.rs:1597-1600; scripts/check-release-delegate.sh:8-13
- Most v1 local-state persistence call sites are `use-node` only, so the offline UI keeps that state in SNAPSHOT and the item writers are exercised offline only by unit tests. ui/src/app.rs:6038; ui/src/local_state.rs:1007-1016; ui/src/app/identity_store.rs:842-899
- Tests are split by feature set: default `cargo test -p freenet-email-ui --lib` and `cargo make test-ui-offline` (--no-default-features --features example-data,no-sync), which adds everything that drives the real delegate in-page. Makefile.toml:907-919; ui/src/mail_delegate/ops.rs:628; ui/src/local_state/items.rs:1160
- Change costs: a protocol change needs a golden append and a candidate rebuild, which changes the delegate key. Guards string-match source lines at ui/src/app/identity_store.rs:152, 807-808, 952 and ui/src/jobs.rs:35, 106-107 (scripts/check-guards-fail.sh:1193, 1318, 1516, 1707-1719). Exhaustive matches at ui/src/mail_delegate/mod.rs:266-291 and ui/src/local_state/items.rs:777-793 must be extended for new Reply or ItemKind variants.

### Docs or comments that disagree with the code

- **[low]** Claim (code comment, ui/src/local_state.rs:4-12): The local-state snapshot is loaded once per page and scoped by alias string (AliasState keyed by alias). **Reality:** It is reloaded on every load_identities call (migration walk, confirm, repairs) and keyed by `@<address hex>`, never by alias. Evidence: ui/src/local_state.rs:226-239; ui/src/app/identity_store.rs:813-875, 294, 317, 749; ui/src/app/migration_view.rs:138-167
- **[low]** Claim (code comments, ui/src/local_state.rs:42-80): Optimistic stashes and draft tombstones are held until the delegate echoes a snapshot that confirms them, then dropped. **Reality:** replace_snapshot is test-only; production installs only through merge_loaded with confirm_discards=false, so draft tombstones are never dropped and the dozen 'stale echo' tests exercise a test-only entry point. Sent/kept/archived stashes are instead drained on ack. Partly admitted at lines 28-33. Evidence: ui/src/local_state.rs:341, 363-386, 940-958
- **[medium]** Claim (code comment, ui/src/mail_delegate/node.rs:113-115): A failed registration attempt is retried by the next request. **Reality:** Only a failed send is retried. An attempt that was sent ends in Done whether the node acknowledged it, stayed silent for 10 s, or refused it. Evidence: ui/src/mail_delegate/node.rs:151-158, 184-199
- **[medium]** Claim (scripts/check-release-delegate.sh:15): Releases before 0.3.0 do not ship the mail delegate, so the release check passes them. **Reality:** The default UI build already embeds the candidate delegate (built with v1-transition) while the workspace version is 0.2.7. Only .release-freeze stops a 0.2.x release from shipping it. Evidence: ui/src/mail_delegate/node.rs:19-22
- **[low]** Claim (code comment, protocol/src/lib.rs:152): Draft item ids are 16 bytes, a content hash. **Reality:** Drafts and sent messages use the 16 bytes of their UUID; only kept/archived/flags use a content hash. Evidence: ui/src/local_state/items.rs:10-12, 109-111
- **[low]** Claim (code comment, ui/src/mail_delegate/ops.rs:174-175): A full identity is about 17k items at 256 per page. **Reality:** The count predates the inbox v2 kinds (PassLedger and Inbound at 4096 each, HeldPass at 2048), so a full identity can be roughly 27k items. The 1000-page bound still covers it. Evidence: delegates/mail-delegate/src/store.rs:224-240
- **[low]** Claim (code comment, ui/src/mail_delegate/local.rs:1-5): The offline build runs the real delegate in the page so the UI exercises the same protocol, origin scoping and compare-and-swap rules it meets on a node. **Reality:** True for identities, contacts and inbox v2. For v1 local mail state the offline UI does not call the item writers (they are gated on use-node); only unit tests do. Evidence: ui/src/app.rs:6038; ui/src/local_state.rs:1007-1016; ui/src/app/identity_store.rs:842-899
- **[low]** Claim (AGENTS.md, 'Sender-side recipient hash' paragraph): The rehydrate and CreateContact prime GETs are in ui/src/api.rs at about lines 1442 and 2540. **Reality:** They are at api.rs:420-430 (prime_contact_inbox) and api.rs:2277; line 1442 is now inside set_inbox_state. Evidence: ui/src/api.rs:420
- **[low]** Claim (AGENTS.md, mail delegate 'Consent' bullet): Consent: exports prompt unless always-allowed; deleting an identity prompts every time. **Reality:** Incomplete rather than wrong: RepairIndex also prompts every time and the client treats it as a consented, long-timeout op. Evidence: ui/src/mail_delegate/mod.rs:81-112
- **[low]** Claim (.release-freeze (repo root)): The background jobs and inbox v2 are still to land. **Reality:** Both are in the tree and described as present by AGENTS.md: the page fallback loop and the v2 UI modules exist. Evidence: ui/src/jobs.rs:102
- **[low]** Claim (code comment, ui/src/app/login.rs:64-66): The AFT tier is hashed into the identity's InboxParams, so changing it rotates the contract id. **Reality:** InboxParams holds only pub_key. The field is also hard-coded to Min10 whenever an identity is rebuilt from the delegate's record. Evidence: wire-v1/src/inbox.rs:37-39
- **[low]** Claim (published-contract/candidate/README.md, file table): inbox_v2 candidate: nothing embeds it yet. **Reality:** The UI embeds it. Outside this subsystem, noted in passing. Evidence: ui/src/v2/mod.rs:27

### Suspected issues (leads from reading, not confirmed)

- **A registration the node refuses is never retried in the page session** (confidence: medium). ensure_registered marks REGISTRATION Done once the request was sent and the 10 s ack wait ended, whatever the node answered. The delegate-error path in api.rs ends pending calls but does not reset the state. If the node refused or lost the registration, every later call would end as Unknown until the page is reloaded. Whether this can happen depends on how freenet-core reports a failed RegisterDelegate, which is not in this tree. Evidence: ui/src/mail_delegate/node.rs:151-158
- **Self-healed own inbox may be re-published with the default AFT policy** (confidence: medium). After a reload an Identity is built by from_info with required_tier Min10 and the default max age. The #288 own-inbox NotFound self-heal builds the inbox state from identity.required_tier and max_age_secs through identity_store::initial_inbox_state, so a user who chose another tier could get it silently reset. I found no place that restores the tier from stored settings, but did not trace every policy-update path. Evidence: ui/src/app/login.rs:93-94; ui/src/api.rs:3668-3672
- **A later load can transiently revert an in-flight draft or settings change** (confidence: medium). merge_loaded replays only changes journaled since that load began, and there is no optimistic stash for drafts or settings. A write still in flight when a migration-, confirm- or repair-triggered load_identities starts can be missing from the loaded state and so reverted in SNAPSHOT until the next edit or reload. The stored copy is not affected. Evidence: ui/src/local_state.rs:303-342
- **items::load writes page caches outside the journal** (confidence: medium). load sets DRAFT_RECIPIENTS, SEEN_DRAFTS and UNREAD_IMPORTS directly while it runs. A recipient the user chose during the load can be overwritten by the stored one. The send path then refuses a mismatch rather than misdelivering, so the effect is a spurious refusal. Evidence: ui/src/local_state/items.rs:868-869, 910
- **An alias shaped like a state key is treated as one** (confidence: medium). state_key returns any string of the form @ plus 48 hex digits unchanged, as if it were an address key. The delegate accepts such an alias (up to 64 chars, no control chars) and the create form only trims. An identity so named would have its local state filed under the address its name encodes. Evidence: ui/src/local_state.rs:258-261
- **Keyless delegate errors are attributed to mail calls while other delegates are in flight** (confidence: low). may_be_mail_delegate_error is passed only whether a token-generator call is in flight. A keyless execution error while a Ghost Key delegate call is waiting would end every non-consented mail call as Unknown. Evidence: ui/src/api.rs:2686-2691
- **Page-lifetime caches that are never drained** (confidence: high). DELETED_MESSAGES has no drain-on-ack counterpart to the sent/kept/archived stashes, forget_key does not clear UNREAD_IMPORTS for a deleted identity, and FULL_IDS only grows. The effect is small memory growth and a deletion re-applied on each later merge. Evidence: ui/src/local_state.rs:527-538
- **mark_read reports failure after the read mark was stored** (confidence: medium). When the kept copy is refused (for example too large), mark_read returns Err after the flag write succeeded. The user is told the mark may not persist although it did, and the PENDING_KEPT body stays in memory for the session because the drain runs only on success. Evidence: ui/src/local_state/items.rs:511-537
- **Stale v2 notes for a contact removed during the startup load** (confidence: low). contact_store::decode inserts v2 notes as a side effect before unchanged_since_load_began filters out contacts the page changed. A v2 contact removed while the load runs can leave notes behind, so the same key re-imported as a legacy contact would be routed by v2. Evidence: ui/src/app/contact_store.rs:102-104
- **Adjacent: a node disconnect panics the loop that delivers delegate answers** (confidence: high). Outside the owned paths, but it bounds everything here: the node loop hits todo!() on RequestError::Disconnect, so no pending call can be answered afterwards. Evidence: ui/src/api.rs:2697-2699

### Open work

- Remove the transitional v1 AFT path: ops::v1_aft_params and its use in identity publish, required before a delegate without v1-transition can be frozen. ui/src/mail_delegate/ops.rs:528-543; ui/src/app/identity_store.rs:1122-1138
- The mail delegate WASM is still a candidate; v0.3.0 must freeze it under published-contract/wasm/v2/ and repoint the includes. ui/src/mail_delegate/node.rs:17-22
- JSON profile export has no UI; export_profile is dead code. ui/src/mail_delegate/ops.rs:554-561
- True unarchive is wired but unused until #60. ui/src/local_state.rs:701-703; ui/src/local_state/items.rs:563-568
- Op::ExportForSuccessor has timeout and consent handling but no ops wrapper and no UI caller. ui/src/mail_delegate/mod.rs:85, 108
- Lost-create recovery matches by alias; a per-create nonce in the protocol would close the remaining ambiguity at the cost of a delegate change. ui/src/mail_delegate/ops.rs:77-84
- No unit test covers the wasm-only timers or a call dropped mid-wait; the test builds have no transport that pends. ui/src/mail_delegate/mod.rs:297-310, 454-458; ui/src/mail_delegate/node.rs:166-175, 185-194
- Documented limit: a timed-out call stays queued in the node and may still run; the page only reports the node as busy. ui/src/mail_delegate/mod.rs:304-308
- Documented limit: the recipient policy cache is in-memory only and re-primed on each load. ui/src/contact_tier_cache.rs:14-15
- Documented gaps in Sent delivery tracking under #58 (rows that can stay Pending). wire-v1/src/local_state.rs:25-27
- Kept copies do not persist threading metadata; such rows group by heuristic. ui/src/app.rs:1303-1305
- Limit by construction: UNREAD_IMPORTS is filled only at load, so an import the delegate makes while the page is open is known as unread only after the next load. ui/src/local_state.rs:36-40; ui/src/local_state/items.rs:893-910

### Open questions

- How does freenet-core answer a failed RegisterDelegate, and an ApplicationMessages request for a delegate that is not registered (an error naming the key, or a keyless one)? That decides whether the registration weak spot can leave a page stuck until reload.
- Is the #288 own-inbox self-heal meant to re-publish with Min10 and the default max age, or is the user's tier restored somewhere I did not find?
- What is the intended order for v0.3.0: remove the UI's V1AftParams use (identity publish and the v1 send path) first, or change the release guard? As written the two cannot both hold.
- Where does the '8 queued page calls' figure in AGENTS.md come from? Nothing in this tree sets it; presumably it is a freenet-core constant.
- Are repeated load_identities runs (after the migration walk, confirm, repairs) meant to be safe against draft and settings writes still in flight, or is the transient revert accepted?
- Should the page learn of a background import that lands while it is open (refresh UNREAD_IMPORTS and the kept copies), or is waiting for the next load the intended behaviour?

### Coverage

Read in full: `/home/user/mail-freenet/ui/src/mail_delegate/mod.rs`, `/home/user/mail-freenet/ui/src/mail_delegate/ops.rs`, `/home/user/mail-freenet/ui/src/mail_delegate/local.rs`, `/home/user/mail-freenet/ui/src/mail_delegate/node.rs`, `/home/user/mail-freenet/ui/src/local_state.rs`, `/home/user/mail-freenet/ui/src/local_state/items.rs`, `/home/user/mail-freenet/ui/src/jobs.rs`, `/home/user/mail-freenet/ui/src/app/identity_store.rs`, `/home/user/mail-freenet/ui/src/app/contact_store.rs`, `/home/user/mail-freenet/ui/src/contact_tier_cache.rs`, `/home/user/mail-freenet/protocol/src/lib.rs`, `/home/user/mail-freenet/ui/src/lib.rs`, `/home/user/mail-freenet/ui/src/log.rs`, `/home/user/mail-freenet/ui/src/rng.rs`, `/home/user/mail-freenet/ui/src/pending_saves.rs`, `/home/user/mail-freenet/ui/Cargo.toml`

Skimmed: `/home/user/mail-freenet/ui/src/api.rs (delegate response and error routing, contacts load, identities_loaded, self-heal, register constants)`, `/home/user/mail-freenet/ui/src/app.rs (merge_inbox_list, autosave and draft delete, store_then and confirmations, send recipient check)`, `/home/user/mail-freenet/ui/src/app/login.rs (Identity struct, from_info, upsert, create-form alias check)`, `/home/user/mail-freenet/ui/src/inbox.rs (select_migrate_from, stranded sends, PENDING_SENT_TIMEOUT_SECS)`, `/home/user/mail-freenet/ui/src/app/address_book.rs (StoredContactKeys only)`, `/home/user/mail-freenet/ui/src/app/migration_view.rs (after_listing reload calls)`, `/home/user/mail-freenet/ui/src/v2/owner.rs (publish and RunJobs)`, `/home/user/mail-freenet/ui/src/v2/flows.rs (module doc, test names, RunJobs test)`, `/home/user/mail-freenet/ui/src/migration/mod.rs (PageStore)`, `/home/user/mail-freenet/ui/src/legacy_registry.rs (candidate hash test)`, `/home/user/mail-freenet/wire-v1/src/local_state.rs (lines 1-560: state and settings types)`, `/home/user/mail-freenet/wire-v1/src/inbox.rs (InboxParams only)`, `/home/user/mail-freenet/delegates/mail-delegate/src/store.rs (lines 30-290: caps, key layout, page access)`, `/home/user/mail-freenet/delegates/mail-delegate/src/handlers.rs (dispatch, item operations, v1_aft_params)`, `/home/user/mail-freenet/delegates/mail-delegate/src/watch.rs (lines 296-435: flags, message id, import)`, `/home/user/mail-freenet/Makefile.toml (test and e2e tasks)`, `/home/user/mail-freenet/Cargo.toml`, `/home/user/mail-freenet/rust-toolchain.toml`, `/home/user/mail-freenet/.release-freeze`, `/home/user/mail-freenet/published-contract/candidate/README.md`, `/home/user/mail-freenet/scripts/check-release-delegate.sh (lines 1-80)`, `/home/user/mail-freenet/scripts/check-guards-fail.sh (guards touching identity_store.rs and jobs.rs)`, `/home/user/mail-freenet/scripts/check-release-scrub.sh (marker checks)`, `/home/user/mail-freenet/.github/workflows/build.yml (step list)`, `/home/user/mail-freenet/.github/workflows/e2e-background.yml (path filters)`, `/home/user/mail-freenet/protocol/src/golden_v1.txt (first lines, line count)`, `/home/user/mail-freenet/ui/tests/mail-delegate-identity.spec.ts (header and test names)`, `/home/user/mail-freenet/ui/tests/export-permission.spec.ts (header and test names)`, `/home/user/mail-freenet/ui/tests/background-proof.spec.ts (header)`, `/home/user/mail-freenet/ui/tests/repro-106-107.spec.ts (header)`

Skipped: `None under the owned paths: all ten files were opened and read in full, test modules included.`
