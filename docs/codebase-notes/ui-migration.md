> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# ui-migration: v0.2.x delegates -> mail delegate, and scrub

Read at upstream commit 2358b59. Nothing was built or run. All paths are relative to the repo root.

## 1. Purpose and place in the system

A v0.2.x user's identities and contacts live in the `identity-management` delegate, and drafts, sent, kept, archived, read marks and settings in `mail-local-state`. Both are keyed by `blake3(code_hash ‖ params)`, so the mail delegate starts empty. This subsystem runs in the page: it asks each predecessor one read, maps the reply by content to `(key, value)` pairs, imports each through the mail delegate's ordinary ops, and (in a scrub build) later deletes the old copy.

The walk order, per-predecessor markers and the journal state machine come from the crates.io crate `freenet-migrate` (`=0.8.0`, `ui/Cargo.toml:48`; `Cargo.lock:1665-1668`; repo github.com/freenet/freenet-migrate; requires stdlib 0.12.x). Its build half `freenet-migrate-build = "0.2.0"` (`ui/Cargo.toml:130`, a caret requirement, pinned only by the lockfile) parses and validates the registries and generates consts. Everything else is local: transport, mapping, the never-overwrite writer, cadence, hold-back, alias resolution, orphans, ledger, scrub.

Entry: `identity_store::load_and_report` -> `migration_view::after_listing(NodeLegacy, NodeInbox)` (`ui/src/app/identity_store.rs:783-799`, `ui/src/app/migration_view.rs:129-171`). Per load: list identities -> `migration::run` -> reload the list if anything arrived -> `scrub::run` -> `confirm_by_inbox` -> refresh the notice.

## 2. Walkthrough

### ui/src/migration/mod.rs (526)
- `IDENTITY_GENERATION_OFFSET = 1_000_000` (`:91`) is added to identity rows, so the crate (newest generation first) imports every identity row before any local-state row. `lineage()` (`:96-107`) has four rows today: local state 205, 206; identity 1000205, 1000206.
- `policy()` = `UnionAllGenerations` with its ack (`:109-113`); authorization is `MigrationAuthorization::app_author_ack()` (`:368`): the import itself asks the user nothing.
- Cadence (`:119-151`): `ABSENT_WALKS_BEFORE_BACKING_OFF = HOLD_BACK_LOADS + 1 = 4`, `BACKED_OFF_ONE_LOAD_IN = 10`, `SILENT_WALKS_BEFORE_SLOWING = 6`, `SLOWED_ONE_LOAD_IN = 4`. `walk_this_load(status, roll)`: `Complete` never walks; every pending row typed "not installed" 4 walks running -> walks iff `roll % 10 == 0`; every pending row not `Started` and (absent streak >= 4 or attempts >= 6) -> iff `roll % 4 == 0`; anything else (an import `Started`, `NotStarted`, `Unreadable`) walks every load. `roll` is one random byte (`migration_view.rs:137`).
- `settled()` (`:182-187`): only `JournalStatus::Complete` is finished; the wildcard arm retries.
- `WalkOutcome` (`:190-207`) carries the status, the crate's report, counts, `silent_rows`, `released_markers`, `read_rows` (what each row mapped to) and `landed` (pairs written verbatim). `unfinished()` (`:157-165`).
- `ONE_WALK` (`:209-214`): a thread-local async mutex held for a whole walk and a whole scrub (`scrub.rs:799-800`).
- `run()` (`:292-409`): `begin_migration_journal` (failure returns `status: None`, nothing imported); stamp `Ledger.migration_started_ms` once from the page clock (`:326-337`); return early if not walking (held markers released only when `Complete`, `:338-351`); build `LegacyAdapter` from the journal's pending rows, the ledger's `absent_answers` and kept alias tables; `migrate_delegate_secrets_journaled`; write the ledger (per-row absent streak +1 or reset, `silent`, `rejected`, `:375-390`); `release_held_markers` (`:261-282`).
- `confirm_by_inbox` (`:426-457`): an unconfirmed identity with a v1 inbox under any registry hash or the current one (`v1_inbox_hashes`, `:413-421`) gets `Op::ConfirmIdentity`.
- `status_report` JSON (`:461-496`), `status_line` (`:499-526`).

### ui/src/migration/legacy_delegates.rs (1055)
- `MAX_LEGACY_REPLY_BYTES = 64 MiB` (`:40`; larger = `NoAnswer`, `:705-707`), `EMPTY_REPLY` (`:44`), `HOLD_BACK_LOADS = 3` (`:53`), alias 64 chars, description 1024 (`:55-56`).
- `LegacyReply { Answer, NotInstalled, NoAnswer }` (`:61-71`); `LegacyTransport::ask` (`:75-82`). Params are `include_bytes!` of `published-contract/wasm/v1/identity-manager-params` and `mail-local-state-params` (`:84-87`).
- `PairKey` (`:106-123`), encoded as text (`:135-183`): `id/<addr>`, `ct/<addr>`, `dr|se|kp|ar/<owner>/<id16>`, `fl/<addr>`, `si/<addr>`, `sg`, `lo/<id16>`, `rj/<id16>`.
- Ids: `id16` = first 16 bytes of blake3 over length-prefixed parts (`:326-335`); draft/sent id = the UUID's bytes (`text_id`, `:340-342`); kept/archived id = `id16(json body)`, since their v1 key is an inbox position (`:526-537`).
- `map_identities` (`:384-449`): decodes `wire_v1::IdentityManagement`; an empty one is `Err`. An identity's `key` is JSON `{ml_dsa_seed (32), ml_kem_seed (64)}`; its address is derived from the ML-DSA-65 key; the four v0.2 contract-marker fields ride along (`MigratedIdentity`, `:188-195`, no `Debug`). Bad seeds or an undecodable contact become a `Refused` pair and `undecodable += 1`. Contacts go through `contact_store::encode_keys`. Also returns alias -> address.
- `map_alias` (`:507-574`): messages become pairs whose value is the re-serialised wire-v1 struct; over `MAX_ITEM_BYTES` (256 KiB) -> refused. `deleted` marks are counted, never imported (`:538`). Read marks: one `Flags` pair, emptied when a newer local-state row holds the alias or cannot be read (`:540-561`). Non-default settings -> `IdentitySettings`. With no owner everything becomes `Orphan` pairs.
- `map_local_state` (`:582-606`): one alias that cannot be placed fails the whole row; zero pairs is `Err`.
- `AliasTables::resolve` / `filed_under` (`:259-284`): the row's own release table is required; an unresolved alias is orphaned only when its own identity row's absence stands.
- `LegacyAdapter` (`:610-958`) implements `PredecessorSecretsIo` (`:1012-1055`). One round trip per row per adapter (`reply`, `:685-713`; `GetIdentities` or `GetAll`). `absence_stands` (`:719-727`): pending, import never started, `absent_answers >= 3`. `held_back_by` (`:882-898`): a newer row of the same family that was silent or unusable this run, with `attempts < 3`. `newest_unsettled_row` (`:736-754`), `alias_tables` (`:762-796`), `newer_aliases` (`:856-875`), `silent_rows` (`:934-944`). Scrub uses `answer`, `reread`, `send`, `holds_nothing` (`:813-853`). `fetch_secrets` records `(key, value hash, value if settings)` per row or marks the row unusable.

### ui/src/migration/successor.rs (918)
- `CAS_ATTEMPTS = 4` (`:36`). Traits `MailStore` (`:39-41`) and `V1InboxReader` (`:45-52`).
- `MigrationCounts` (`:55-122`): written items add up; `refused`, `at_cap`, `identity_gone`, `flags_skipped`, `deleted_marks` keep the maximum.
- `ArrivedIdentities` (`:128-161`), `merge_fields` (stored wins, `:166-186`), `MergedFlags` OR-merge (`:190-205`), `adopt_legacy_markers` (`:217-270`), `StoredMarker` (`:274-308`).
- `get_item`, `put_item`, `merged_put` (`:343-391`).
- `MailSuccessor` (`:393-801`): `failed` (`:429-447`), `is_deleted` (`:453-466`), `create` (`:543-562`), `import_identity` (`:564-626`), `adopt_markers` (`:628-652`), `merge_settings` (`:654-674`), `import_flags` (`:678-746`), `apply` (`:764-800`). Implements `SuccessorSecretsIo` (`:803-884`) and `JournaledSuccessorIo` (`:886-918`); `import_revision` is not overridden (crate default 0).

### ui/src/migration/ledger.rs (181)
`Ledger` (`:17-54`, JSON): `totals`, `notice_dismissed`, `absent_answers`, `silent`, `rejected`, `migrated`, `migration_started_ms`, `snapshot_rows`, `moved`, `deleted_identities`, `scrubbed`, `scrub_empty`, `scrub_blockers`. `read_ledger` treats an undecodable body as an error (`:117-123`); `update_ledger` will not write over one (`:127-154`). `list_items` pages 256 at a time (`:156-181`).

### ui/src/migration/orphans.rs (205)
`groups` (`:46-61`); `move_one` (`:117-140`): item, then the ledger's `moved`, then the orphan's delete, so an interruption leaves two copies; `create` accepts a conflict only over identical bytes (`:82-93`); `discard` takes only the ids shown (`:187-205`).

### ui/src/migration/node.rs (354, `use-node` only, `mod.rs:64-65`)
`REPLY_TIMEOUT_MS = 15_000` (`:24`). Waiters keyed by `DelegateKey` and `ContractKey` (`:26-31`); replies are matched by key, not by request. `NodeLegacy::ask` sends `DelegateRequest::ApplicationMessages` and never registers the legacy WASM (`:180-212`). `classify_response` (`:35-45`): only a non-empty application message is an answer. `classify_error` (`:53-65`): `DelegateError::Missing(this key)` -> `NotInstalled`; `RegisterError`/`MissingSecret` for it -> `NoAnswer`; a keyless error is taken only when no mail or token call is in flight. `NodeInbox` (`:216-255`) GETs the v1 inbox; `ciphertexts` reads only under the current `INBOX_CODE_HASH` (`:247`).

### ui/src/migration/scrub.rs (807)
`BUILT_IN` = `option_env!("FREENET_MAIL_SCRUB") == "1"` (`:83-86`); `BUILD_MARKER = "freenet-mail:scrub-built-in"` only then (`:90-94`); `EMPTY_LOADS = 2` (`:97`); snapshot entry = `id16(key) ‖ value hash`, 4096 entries per chunk (`:77-80`, `:148-153`); `Blocker` (`:101-136`). `run` (`:777-788`) -> `run_built` (`:793-807`) -> `attempt` (`:697-774`). Conditions are in section 4.

### ui/build.rs (155), ui/src/legacy_registry.rs (152), legacy_*.toml
`emit_legacy_registries` (`build.rs:111-148`) runs for every feature set: `Registry::from_path` per file, panic when a file has neither `[[contract]]` nor `[[delegate]]`, then three `codegen()` calls into `$OUT_DIR`. `Registry::validate` (in the build crate) checks hash encodings, duplicate generations and hashes, and `delegate_key == blake3(code_hash ‖ params_hex)`. The frozen-artifact presence check runs only with `use-node` and without `no-sync` (`build.rs:35-107`). `legacy_registry.rs` re-exports the consts (`:24-42`).
Rows: inbox 0 `Ht8rc...`, 205 `9Kuhod...` (`legacy_inbox.toml:17-25`); identity delegate 205 `2gFVM...`, 206 `BbtKh...` (`legacy_identity_delegate.toml:20-32`); local state 205 `2VLTi...`, 206 `5FGNJ...` (`legacy_local_state_delegate.toml:19-31`). `generation = major*10000 + minor*100 + patch`. Both generations of a family share one `params_hex`.

### tools/v1-node-seed (workspace member, `Cargo.toml:17`; main.rs 301)
`v1-node-seed <ws port> [capture file]` registers the frozen WASM with the v0.2.x cipher and nonce (`main.rs:32-39`, `:62-70`), seeds identity `alice` (seeds `[9;32]`, `[10;64]`), contact `bob`, archive at position 4, kept+read at 3, draft `d1`, sent `s1` (`:183-258`), checks read-back and optionally writes both replies as hex. `--expect-cleared` (`:152-181`) fails on an empty answer or any seeded text left.

## 3. Interfaces

- **To the old delegates:** JSON messages from `freenet-mail-wire-v1`: `IdentityMsg::{GetIdentities, DeleteIdentity}`, `LocalStateMsg::{GetAll, DeleteDraft, DeleteSent, DeleteMessage}` (`scrub.rs:561-596`); replies decode as `IdentityManagement` / `LocalState`.
- **To the mail delegate** (`freenet-mail-protocol` ops via `mail_delegate::call`): `ImportIdentity{Seeds, ImportKind::Migration}`, `PutItem`, `GetItem`, `ListItems`, `DeleteItem`, `ListIdentities`, `GetStatus`, `ConfirmIdentity`.
- **Storage** (per origin): journal = `ItemKind::MigrationJournal`, empty id (`successor.rs:326-328`); per-predecessor marker = `MigrationMarker` with the 32-byte delegate key as id (`:330-332`); ledger, snapshot chunks and alias tables are also `MigrationMarker` items with ids `blake3("freenet-mail migration record\0" ‖ name ‖ index)` (`ledger.rs:56-82`). All share the kind's cap of 256 (`delegates/mail-delegate/src/store.rs:40,238`) and are not exported (`store.rs:206-208`). Orphans = `LegacyOrphan` (16-byte id, cap 4096, exported). Contract markers live in each identity's `IdentitySettings` under `"migration"` (`identity_store.rs:541-564`); `pending_held` stops `plan_load` (`identity_store.rs:952`).
- **Node loop hooks** in `ui/src/api.rs`: `offer_error` (`:2675`), `offer_contract_state` (`:2723`), `takes_response`/`offer_response` (`:3417-3419`), `offer_contract_not_found` (`:3599`).
- **Callers in:** `migration_view.rs` (notice, orphans UI, `gate`), `identity_store.rs:152-163` (`record_identity_deleted`), `app/migration_fixture.rs` (offline `?migration-fixture=data|silent|not-installed`, `:20-28`, `:91-97`). Scrub's backup evidence is `last_backup_at`, stamped in `app/login.rs:517-523`.
- **Features and env:** `use-node` (real transport), `example-data` (fixture), env `FREENET_MAIL_SCRUB=1` at build time. The mail delegate is a dev-dependency so tests run its real handlers in every feature set (`ui/Cargo.toml:124-127`).
- **From the crate:** `begin_migration_journal`, `migrate_delegate_secrets_journaled`, `JournalStatus` (`pending_all_absent`, `interrupted_import`, `rejected_items`), `PendingPredecessor{attempts, import, absent_streak}`, `ItemWrite`, `MigrationMarker`, `ProbeVerdict`, `DelegateLineageEntry`. The mail delegate also uses the crate for its own successor export (`delegates/mail-delegate/src/handlers.rs:1704-1741`).

## 4. Invariants and where they are enforced

Row states. *Pending*: listed in `JournalStatus::Incomplete.pending`. *Resolved*: not listed (every pair written or already authoritative, or only permanent rejections). *Held back*: this module returns `Err` from `fetch_secrets` (`legacy_delegates.rs:905-907`). *Standing absent*: `absence_stands` plus an absent or empty answer now.

1. **Silence is never absence; nothing is sealed empty.** `fetch_secrets` never returns `Ok(vec![])` (`legacy_delegates.rs:389-391`, `:602-604`); timeouts and empty replies are `NoAnswer` (`node.rs:35-45`, `:170-174`). Tests `a_silent_get_is_unresponsive_never_absent_never_sealed`, `scenario_an_answered_empty_store_is_never_sealed`, `scenario_a_typed_not_installed_stays_pending_and_is_never_sealed` (`tests.rs:630-686`).
2. **Import never overwrites.** Create-only `PutItem{expected_gen: None}`; a `Conflict` is checked by a read (`successor.rs:543-562`); settings and flags merge with the stored value winning. Tests `scenario_an_import_never_overwrites_what_the_new_delegate_holds` (`tests.rs:852`), `scenario_a_false_conflict_does_not_pass_for_a_stored_item` (`:1471`), `:2788`.
3. **Identities before local state; the offset is fixed.** `mod.rs:86-91`; `the_lineage_has_every_row_and_identities_rank_above_local_state` (`tests.rs:1791`).
4. **Hold-back is bounded at 3 walks; "not installed" never holds.** `legacy_delegates.rs:882-898`; `tests.rs:1050-1098`, `:2647`; numbers pinned by `the_limits_are_the_documented_numbers` (`:2633`).
5. **Orphaning needs a standing absence or a complete set of tables.** `legacy_delegates.rs:259-284`, `:719-727`; `tests.rs:966-1047`, `:2188-2328`.
6. **Permanent rejections** (`successor.rs:429-447`): `BadInput`; `CapReached` (`at_cap`); `Refused` on an owned item (`identity_gone`); `NotFound` on an owned item only if the owner is in `ArrivedIdentities`, `GetStatus.unreadable_identities == 0` and `ListIdentities` lacks it (`:453-466`). Also `Refused` pairs, unknown keys, undecodable values, seeds that do not derive the address (`:570-612`, `:795-798`, `:868-870`). Everything else is retryable. Tests `tests.rs:1487-1588`, `:2808`.
7. **Contract markers come only from the newest release holding the identity, and wait while a newer row is unsettled.** `successor.rs:217-270`, `:628-652`; `mod.rs:261-282`; `tests.rs:1103-1300`.
8. **Journal before any import; one walk per tab.** `mod.rs:309-324`, `:304-305`; `a_journal_that_cannot_be_stored_stops_before_any_import` (`tests.rs:1677`), `:1618`.
9. **Markers never lose `Done` or a data flag.** `successor.rs:282-308`; `a_marker_follows_the_crates_contract` (`tests.rs:1916`).
10. **Scrub deletes only if all of these hold** (`scrub.rs:697-774`), each with a test in `tests/scrub.rs` where it alone is false:
    - the build has the flag (`:783-785`; `a_default_build_cannot_reach_a_delete`, `:749`);
    - the journal was established and stored this load (`:703-709`; `:272`, `:1381`);
    - targets are resolved rows not yet finished; every pending row stands absent and answers so now (`:745-760`; `:283-377`);
    - nothing rejected, no orphans left, every migrated identity protected (`cheap_blockers`, `:436-457`; `:394`, `:431`);
    - each target row is read fresh and maps (`read_rows`, `:464-489`); no `Refused` entry; every owner the rows name has `last_backup_at > migration_started_ms`, or is in `deleted_identities` if not listed (`unprotected`, `:414-432`; `:474-543`, `:623`, `:1244-1318`); the ML-KEM key the delegate lists equals the one the row's seed gives (`kem_differs`, `:497-513`; `:1272`);
    - every compared pair (not flags) is the current import, or in the row's snapshot, or a moved/discarded orphan (`row_blockers`, `:540-556`; `:544-600`, `:778-1104`); a missing chunk blocks (`:642`);
    - local state first; an identity row's alias table is kept before its first delete (`:622-638`, `:660-669`); each row is journaled in `scrubbed` before deletes (`:670-673`), re-read and byte-compared just before the first delete (`:609-611`; `:1187`); each delete must be acknowledged by an empty reply (`:612-617`); a row is finished after 2 loads with nothing left to delete (`:681-691`; `:1137-1185`); identity rows wait while any local-state row is unfinished (`:662-665`, `:692`).
11. **Registries are append-only and pinned.** `registries_pin_exact_rows`, `the_newest_rows_are_the_frozen_v1_delegates`, `the_generation_in_use_is_not_a_predecessor` (`legacy_registry.rs:66-151`).

## 5. State

- **Frozen:** the v0.2.x delegates and params (`published-contract/wasm/v1/`), the registry rows, the wire-v1 types.
- **Not yet released, frozen on first release:** `IDENTITY_GENERATION_OFFSET`, the `PairKey` text, id derivations (`id16`, `text_id`), `StoredMarker` and `Ledger` JSON, the reserved item ids, the snapshot entry layout, and the lineage plus policy the journal is bound to. Users will hold all of them in stored records.
- **Candidate:** the mail delegate the walk writes to (`published-contract/candidate/`, referenced by `legacy_registry.rs:142-144`).
- **Releases are blocked** by `.release-freeze`, which names "the rehearsal against real v0.2.x data" (`.release-freeze:5-8`).

## 6. Tests

- `ui/src/migration/tests.rs` (91 tests): the crate's real walk (`run`, roll 0, `:450-452`) over a scripted `Node` (`:310-414`, no script = "not installed"; it applies legacy deletes to its JSON) into the real delegate handlers (`mail_delegate::handlers::process`, `:216-234`) with fault injection (`Faults`, `:114-129`; `die_after`; `blind`; `hide_identity`). Inputs are `wire-v1/src/golden_v1.txt` samples edited as text, plus `captured_v0_2_7.txt` (`:2832-2917`). Covers every row state, interruption at every write (`:719`), cadence, markers, merges, orphans, confirm-by-inbox, and source-text checks that share and send pass the gate (`:2588`).
- `ui/src/migration/tests/scrub.rs` (58 tests): `usual_node` (one generation installed), `settle`, `ready`, `scrub_all` (3 loads); asserts on the legacy deletes actually sent.
- `node.rs` (5 tests, default features only), `legacy_registry.rs` (3).
- Commands: `cargo test -p freenet-email-ui --lib migration` (`scripts/check-guards-fail.sh:128-129`); `cargo make test` also runs the offline feature set (`Makefile.toml:916-919`), where `node.rs` is not compiled. Do not export `FREENET_MAIL_SCRUB=1`: two tests assert the default (`tests/scrub.rs:742-767`).
- `cargo make test-e2e-legacy-migration` (`Makefile.toml:1289-1337`): wipes and starts the iso nodes, publishes with `FREENET_MAIL_SCRUB=1 FREENET_MAIL_SCRUB_ISO=1`, runs `check-release-scrub.sh`, seeds, runs `ui/tests/legacy-migration.spec.ts` (`FREENET_LEGACY_SEEDED=1`, chromium), then `v1-node-seed --expect-cleared`. Needs `freenet`, `fdev`, `dx`, Playwright, python3. CI: `.github/workflows/e2e-real-node.yml:181` (pull requests, tags, nightly).
- `cargo make capture-v0-2-7-replies` (`Makefile.toml:1265-1287`) regenerates the captured file. Offline screens: `ui/tests/migration-ui.spec.ts` (7 tests).

## 7. Changing it safely

- **Guards anchor on exact source text.** About 100 mutations in `scripts/check-guards-fail.sh:1007-1600` edit lines of these files (legacy_delegates 23, mod 9, node 2, orphans 4, scrub 46, successor 15) and require named tests to fail. Reformatting or renaming an anchored line breaks its guard.
- A new registry row: append to the TOML, update `registries_pin_exact_rows` in the same commit; a delegate row needs `params_hex` that derives its key. The walk then reads it with the family's v1 params (`legacy_delegates.rs:692-703`), so a row with other params needs code.
- Never change a generation, the offset, or the `PairKey` text after release: journal entries, markers and snapshots are keyed by them.
- Returning `already_authoritative` for a failed write loses data silently (the crate's own warning); use `retryable` when unsure.
- A new item kind to migrate needs: a `PairKey`, a mapping, an `apply` arm, `is_compared`/`stored_item` in scrub, a delete in `row_deletes` if wire-v1 has one, and a `destination` for orphans.
- A new golden sample comes from `wire-v1/capture`, not from today's types (AGENTS.md).
- Nothing here forces a candidate rebuild unless the delegate, `protocol/` or `wire-v1/` change.

## 8. Disagreements between docs/comments and code

1. `.release-freeze:5-6` says scrub is still to land; it is in the tree.
2. `legacy_registry.rs:8-9` says the inbox list "is not read at runtime yet"; `mod.rs:414` reads it on every load for `confirm_by_inbox`.
3. `legacy_inbox.toml:6-7` (and both delegate registries, `:6-8`) say `legacy_registry.rs` "refuses a registry that contains" the generation in use. It is a unit test (`legacy_registry.rs:135-151`); the build accepts such a registry.
4. AGENTS.md ("Append the previous hash to the slice (and to `legacy_inbox.toml`)") implies the two inbox lists agree. The registry has two rows; `ui/src/inbox.rs:71-76` has one (`Ht8rc...`). The v0.2.5 hash `9Kuhod...` is tried by the contract migration only when an identity's imported marker names it.
5. AGENTS.md: "local state waits until the identity row of the same generation has answered with something". The code also releases it, as orphans, once that row has stood absent for more than 3 loads (`legacy_delegates.rs:281`, `mod.rs:38-44`).
6. AGENTS.md and `scrub.rs:10` say scrub "is compiled in only with" the flag. The code is always compiled; a const gates `run` (`scrub.rs:783-785`) and tests call `run_built` in a default build.
7. `ui/build.rs:11-15` calls the required v1 files "the WASM the UI embeds". The two legacy delegate WASM files are embedded only under `#[cfg(test)]` (`api.rs:1310-1319`, `local_state.rs:731-739`); a release build embeds only their params.
8. AGENTS.md and `tests.rs:4-6`: old-format inputs come "only" from `golden_v1.txt`. Tests also use `captured_v0_2_7.txt` and typed literals for empty states (`tests.rs:2651-2652`).

## 9. Suspected weak spots

1. **A cap is a permanent rejection** (`successor.rs:433-436`), against the crate's rule that `Permanent` must hold across app versions. The item stays in the predecessor, the row resolves, `Blocker::Rejected` then stops scrub for good, and nothing retries after the user makes room: `import_revision` stays 0. Deliberate (`tests.rs:1487`), but there is no recovery path. High confidence in the reading.
2. **Read marks are dropped for good when the v1 inbox cannot be fetched on the importing load.** `import_flags` returns `already_authoritative` (`successor.rs:705-708`), and `NodeInbox::ciphertexts` asks only under the current hash with a 15 s timeout (`node.rs:247`). A slow node, or an inbox still under an older hash, loses them. Medium.
3. **Scrub can block for good with two generations installed:** an older row that resolves on a later walk than its newer sibling, holding an older copy of a shared key, gets no snapshot entry (`scrub.rs:335-346`: `newer` only sees rows read in the same walk) and then counts as `NotImported`. Nothing is deleted, so it is safe, but never clears. Medium.
4. **A row whose clearing began and that then answers "not installed" is `RowFailed` on every load** (`read_rows` accepts it at `scrub.rs:478-483`, `clear_row` needs an `Answer` at `:604`), so later identity rows never clear. Medium-low.
5. **Replies are matched by delegate key only** (`node.rs:4-7`). A reply arriving after its 15 s timeout can be taken by the next request to that key. The scrub paths fail closed on a mismatch, so the likely cost is a wasted load. Low.
6. `confirm_by_inbox` makes up to three sequential GETs per unconfirmed identity on every load (`mod.rs:436-443`). Low.

## 10. Open work

- The rehearsal against real v0.2.x data: nothing in the tree records or automates one. The real-node test uses seeded data from the 206 delegates only; no 205 WASM is in the tree, so a 205 row is never read from a real delegate.
- Documented limits: settings and most read marks are never deleted from the old delegates (`scrub.rs:51-52`); `deleted` marks are not imported; a deleted item can return while a row is retried, and an identity deleted in a newer release can return from an older one (`mod.rs:20-27`, shown to the user at `migration_view.rs:20-22`); the journal is normally never `Complete`.
- No TODO or FIXME markers in the owned files.

## 11. Questions the tree does not answer

1. What the rehearsal must cover and who has real v0.2.5/0.2.7 node data to run it against.
2. Whether a production node answers `Missing` for a delegate whose WASM was unregistered while its secrets remain, and whether the UI should register the frozen WASM to reach them (it never does, `legacy_delegates.rs:73-74`).
3. Whether v0.2.5 delegate replies match the wire-v1 types exactly (only 206-era bytes are captured).
4. Which `FailureCode` the delegate gives for a byte-budget refusal, and whether `Refused` on an owned write can mean anything but "identity being deleted" (both decide permanence here; not read in the delegate).
5. How the crate counts `attempts` for a row this module holds back; the crate's 15.7k-line driver was read only at its public types and docs.

## Appendix: files

Read in full: `ui/src/migration/{mod,legacy_delegates,successor,scrub,node,ledger,orphans}.rs`, `ui/src/legacy_registry.rs`, `ui/build.rs`, the three `legacy_*.toml`, `tools/v1-node-seed/{Cargo.toml,src/main.rs}`, `.release-freeze`, `ui/tests/legacy-migration.spec.ts`.
Skimmed (docs, every test name, harness and selected bodies): `ui/src/migration/tests.rs`, `ui/src/migration/tests/scrub.rs`, `captured_v0_2_7.txt` (header), `ui/src/app/migration_view.rs:1-260`, `ui/src/app/migration_fixture.rs`, parts of `ui/src/app/identity_store.rs`, `ui/src/api.rs`, `Makefile.toml`, `scripts/check-guards-fail.sh`, `delegates/mail-delegate/src/store.rs`, and the registry copies of `freenet-migrate` 0.8.0 (`lib.rs`, `lineage.rs`, parts of `journal.rs` and `delegate_migrate.rs`) and `freenet-migrate-build` 0.2.0 (`lib.rs`, item list of `registry.rs`).

---

## Appendix: the reader's structured summary

Page-side migration of a v0.2.x user's data (identity-management and mail-local-state delegates) into the mail delegate, plus scrub, which later deletes the old copy. The walk order, per-predecessor markers and the journal come from the crates.io crate freenet-migrate (=0.8.0); freenet-migrate-build (0.2.0) validates the three legacy_*.toml registries in ui/build.rs and generates consts. Everything else is local: the node transport, content-based mapping to (key, value) pairs, a never-overwrite writer over ordinary delegate ops, walk cadence, hold-back of older rows, alias resolution, orphans, a JSON ledger, and scrub with a long list of preconditions.

State: the code is complete and heavily tested (91 walk tests and 58 scrub tests against the real delegate handlers, a real-node e2e seeded by tools/v1-node-seed, about 100 mutation guards anchored on exact source lines). None of it has shipped: releases are blocked by .release-freeze, which still asks for a rehearsal against real v0.2.x data, and nothing in the tree records or automates one. No real v0.2.5 (generation 205) delegate is ever read by a test.

Main leads from reading: a cap refusal is a permanent rejection with no retry path and then blocks scrub for good; read marks are dropped permanently if the v1 inbox GET fails on the importing load; scrub can block forever for users with two generations installed. None of these deletes data.

The map file was written incrementally and is complete on disk, so a restart can reuse it rather than redo this unit.

### Key facts

- freenet-migrate is a crates.io dependency pinned exactly (ui/Cargo.toml:48 `=0.8.0`; Cargo.lock:1665-1668), also used by the mail delegate for its own successor export (delegates/mail-delegate/Cargo.toml:33, src/handlers.rs:1704-1741). freenet-migrate-build is `0.2.0` without `=` (ui/Cargo.toml:130), pinned only by the lockfile.
- The crate supplies the walk driver, markers and journal (begin_migration_journal, migrate_delegate_secrets_journaled, JournalStatus, ItemWrite). This repo supplies both IO adapters: LegacyAdapter implements PredecessorSecretsIo (ui/src/migration/legacy_delegates.rs:1012-1055) and MailSuccessor implements SuccessorSecretsIo and JournaledSuccessorIo (ui/src/migration/successor.rs:803-918).
- Entry point: identity_store::load_and_report -> migration_view::after_listing (ui/src/app/identity_store.rs:783-799, ui/src/app/migration_view.rs:129-171). Order per load: list identities, migration::run, scrub::run, confirm_by_inbox.
- Lineage is four rows: local state 205 and 206, identity 1000205 and 1000206. IDENTITY_GENERATION_OFFSET = 1_000_000 (ui/src/migration/mod.rs:91) makes identity rows import first and can never change, because journal entries record generations.
- Policy is UnionAllGenerations with MigrationAuthorization::app_author_ack (mod.rs:109-113, :368): no user consent for the import, and a deleted item can return from an older generation (disclosed in migration_view.rs:20-22).
- Cadence numbers (mod.rs:119-151, pinned by tests.rs:2633): HOLD_BACK_LOADS 3, back off to 1 load in 10 after 4 walks of typed not-installed, slow to 1 in 4 after 6 unresolved walks; an import under way walks every load.
- A row never resolves as empty: fetch_secrets returns Err for silence, timeout, empty, undecodable, oversized (>64 MiB) and not-installed (legacy_delegates.rs:389-391, :602-604, :705-707), so the journal is normally never Complete; only settled() decides finished (mod.rs:182-187).
- Pair keys are text (`id/`, `ct/`, `dr|se|kp|ar/`, `fl/`, `si/`, `sg`, `lo/`, `rj/`; legacy_delegates.rs:135-183). Kept and archived ids are a hash of the re-serialised JSON body; draft and sent ids are the UUID bytes (:326-342, :526-537). These become frozen on first release.
- Never-overwrite: messages, contacts and orphans are create-only PutItem with a read after any Conflict (successor.rs:543-562); settings merge with the stored value winning (:166-186, :654-674); flags OR-merge (:190-205); identities go through Op::ImportIdentity with ImportKind::Migration (:582-590).
- Permanent rejections are decided in MailSuccessor::failed (successor.rs:429-447): BadInput, CapReached, Refused on an owned item, and NotFound only when the owner is in ArrivedIdentities, GetStatus shows no unreadable identity and ListIdentities lacks it (:453-466). import_revision is left at the crate default 0.
- Storage: journal is ItemKind::MigrationJournal; per-predecessor markers, the ledger, snapshot chunks and kept alias tables are all ItemKind::MigrationMarker items (successor.rs:326-332, ledger.rs:56-82), sharing a cap of 256 and never exported (delegates/mail-delegate/src/store.rs:40, :206-208, :238). Orphans are ItemKind::LegacyOrphan.
- Scrub is gated by a const from option_env!("FREENET_MAIL_SCRUB") (scrub.rs:83-86, :783-785); the marker string `freenet-mail:scrub-built-in` is in the binary only then, and scripts/check-release-scrub.sh reads it from the built WASM.
- Scrub deletes only when: journal stored this load; every pending row stands absent (3 absent answers recorded plus absent now); no rejection, no orphan, no unmappable entry; every identity backed up after migration_started_ms or recorded as user-deleted; ML-KEM keys agree; every compared pair is the current import, in the row's snapshot, or a handled orphan (scrub.rs:697-774).
- Scrub order and resumption: local state before identities, alias table kept before an identity row's first delete, row re-read and byte-compared before the first delete, each delete acknowledged by an empty reply, a row finished only after 2 loads with nothing left to delete (scrub.rs:600-695, EMPTY_LOADS at :97).
- The real transport (node.rs, use-node only) never registers legacy WASM, waits 15 s per request (node.rs:24) and matches replies by delegate key, not request id (:4-7, :26-31). Only DelegateError::Missing for the waiting key is NotInstalled (:53-65).
- ui/build.rs:111-148 validates the registries for every feature set and panics on an empty one; the build crate checks delegate_key == blake3(code_hash ‖ params_hex). legacy_registry.rs:66-151 pins every row in a unit test.
- About 100 guards in scripts/check-guards-fail.sh:1007-1600 mutate exact source lines of ui/src/migration/*.rs (scrub 46, legacy_delegates 23, successor 15, mod 9, orphans 4, node 2); reformatting or renaming an anchored line breaks its guard.
- Tests: `cargo test -p freenet-email-ui --lib migration` (check-guards-fail.sh:128-129) runs the real crate walk against the real delegate handlers (tests.rs:216-234) with roll 0; two scrub tests fail if FREENET_MAIL_SCRUB=1 is exported (tests/scrub.rs:742-767).
- Real-node proof: `cargo make test-e2e-legacy-migration` (Makefile.toml:1289-1337) builds with scrub, seeds via tools/v1-node-seed, runs ui/tests/legacy-migration.spec.ts, then `v1-node-seed --expect-cleared`; CI runs it in e2e-real-node.yml:181.
- Releases are blocked by .release-freeze, which names a rehearsal against real v0.2.x data (.release-freeze:5-8); no script, record or data for one exists in the tree.

### Docs or comments that disagree with the code

- **[medium]** Claim (.release-freeze:5-6): The scrub of what the walk leaves behind is still to land. **Reality:** Scrub is in the tree and wired into every load (ui/src/migration/scrub.rs; called at ui/src/app/migration_view.rs:156). Evidence: ui/src/migration/scrub.rs:777
- **[low]** Claim (code comment, ui/src/legacy_registry.rs:8-9): The inbox list is not read at runtime yet (the shipped inbox is still the v1 one). **Reality:** v1_inbox_hashes reads LEGACY_INBOX_CODE_HASHES on every load for confirm_by_inbox. Evidence: ui/src/migration/mod.rs:414
- **[medium]** Claim (legacy_inbox.toml:6-7, legacy_identity_delegate.toml:7-8, legacy_local_state_delegate.toml:6-7): ui/src/legacy_registry.rs refuses a registry that contains the generation in use. **Reality:** It is a unit test (the_generation_in_use_is_not_a_predecessor). The build accepts such a registry; only `cargo test` catches it. Evidence: ui/src/legacy_registry.rs:135
- **[medium]** Claim (AGENTS.md, 'Chained migration (#251 improvement 2)'): Append the previous hash to the slice (and to legacy_inbox.toml) as part of any inbox bump, implying the in-source slice and the registry hold the same rows. **Reality:** legacy_inbox.toml has two rows (generation 0 and the v0.2.5 hash 9Kuhod...); the in-source LEGACY_INBOX_CODE_HASHES that the contract migration walks has one (Ht8rc...). The v0.2.5 hash is tried only when an identity's imported marker names it. Evidence: ui/src/inbox.rs:71
- **[low]** Claim (AGENTS.md, 'An alias is orphaned only on its own generation's word'): Local state waits until the identity row of the same generation has answered with something. **Reality:** It also stops waiting once that identity row has stood absent or empty for more than HOLD_BACK_LOADS loads: its aliases are then filed as orphans and the rest of the row imports. Evidence: ui/src/migration/legacy_delegates.rs:281
- **[low]** Claim (AGENTS.md 'Scrub (scrub.rs) clears the old copy' and ui/src/migration/scrub.rs:10): Scrub is compiled in only with FREENET_MAIL_SCRUB=1. **Reality:** All scrub code is always compiled; a const from option_env! makes run() return early, and tests call run_built in a default build. Only the marker string depends on the flag. Evidence: ui/src/migration/scrub.rs:783
- **[low]** Claim (code comment, ui/build.rs:11-15 (and the required list at :65-73)): The required v1 files are the contract/delegate WASM the UI embeds via include_bytes!. **Reality:** The identity_management and mail_local_state WASM files are embedded only in #[cfg(test)] modules; a release build embeds only their params files for the walk. Evidence: ui/src/api.rs:1310
- **[low]** Claim (AGENTS.md 'Tests drive the crate's real walk' and ui/src/migration/tests.rs:4-6): Old-format test inputs are taken only from wire-v1/src/golden_v1.txt. **Reality:** Tests also read captured_v0_2_7.txt and use typed literals for empty states. Evidence: ui/src/migration/tests.rs:2832

### Suspected issues (leads from reading, not confirmed)

- **A cap refusal is a permanent rejection with no retry path, and then blocks scrub for good** (confidence: high). MailSuccessor::failed maps CapReached to ItemWrite::permanent. The crate's own rule is that Permanent must hold across app versions and is not for a refusal that can change. The row resolves, the item stays only in the old delegate, Blocker::Rejected stops scrub on every later load, and nothing re-offers the item after the user makes room: import_revision is left at 0. The choice is deliberate and tested (tests.rs:1487), so this is a design gap, not an accident. Evidence: ui/src/migration/successor.rs:433
- **Read marks are lost for good when the v1 inbox cannot be fetched on the importing load** (confidence: medium). import_flags returns already_authoritative when ciphertexts() gives None, so the row can resolve and the marks are never retried. NodeInbox::ciphertexts asks only under the current INBOX_CODE_HASH with a 15 s timeout, so a slow node, or an inbox still under an older hash (a v0.2.5 user before the contract migration ran), drops every read mark. Counted as flags_skipped, pinned by tests.rs:1656. Evidence: ui/src/migration/successor.rs:705
- **Scrub can block forever when an older row resolves on a later walk than its newer sibling** (confidence: medium). write_snapshots records an older row's superseded copy only if a newer row holding the same key was read in the same walk (`newer` looks at outcome.read_rows). If the newer row resolved on an earlier walk, the older copy neither landed nor equals the stored item, gets no snapshot entry, and row_blockers reports NotImported on every load. Nothing is deleted, so it is safe, but the old copy is never cleared. Not covered by a test (the same-walk case is, tests/scrub.rs:876). Evidence: ui/src/migration/scrub.rs:335
- **A row under clearing that turns 'not installed' fails every load** (confidence: low). read_rows accepts a row whose clearing began and that now holds nothing, including a typed not-installed answer, but clear_row needs adapter.answer() to be an Answer and returns None otherwise, giving Blocker::RowFailed each load. clear_rows stops there, so later identity rows are never cleared. Evidence: ui/src/migration/scrub.rs:604
- **Legacy replies are matched by delegate key, not by request** (confidence: low). A reply that arrives after its 15 s timeout can be taken by the next request to the same key (walk read, scrub re-read, or a delete expecting an empty ack). The scrub paths fail closed on any mismatch, so the likely cost is a wasted load, but a stale read could stand in for a fresh one. Evidence: ui/src/migration/node.rs:26
- **confirm_by_inbox costs up to three sequential GETs per unconfirmed identity on every load** (confidence: low). For each unconfirmed identity it asks every registry hash plus the current one in turn, each with a 15 s timeout, on every page load for as long as the identity stays unconfirmed. Evidence: ui/src/migration/mod.rs:436

### Open work

- Rehearsal against real v0.2.x data: named as a release blocker and absent from the tree (.release-freeze:5-8).
- No generation-205 (v0.2.5) delegate WASM is in the tree, so no test reads a real 205 row; the real-node test seeds only the frozen 206 delegates (tools/v1-node-seed/src/main.rs:139-141).
- import_revision is not overridden, so a wrong or outdated permanent rejection cannot be reopened without a code change (ui/src/migration/successor.rs:886-918).
- Documented limit: settings and read marks outside kept and archived messages are never deleted from the old delegates, because wire-v1 has no delete for them (ui/src/migration/scrub.rs:51-52).
- Documented limit: `deleted` marks are never imported (ui/src/migration/legacy_delegates.rs:305-308, :538).
- Documented limit: a deleted item can return while a row is retried, and an identity deleted in a newer release can return from an older one (ui/src/migration/mod.rs:20-27).
- Documented limit: a snapshot written late (the tab died before writing it) records only what still equals the import, so a later user edit can leave scrub blocked (ui/src/migration/scrub.rs:39-42).
- The journal is normally never Complete, so the walk keeps running one load in ten indefinitely (ui/src/migration/mod.rs:31-37, :119-122).
- When the mail delegate is frozen into published-contract/wasm/v2/, the test path at ui/src/legacy_registry.rs:142-144 (candidate/mail_delegate_code_hash) must move with it.
- No TODO or FIXME markers in the owned files.

### Open questions

- What must the rehearsal against real v0.2.x data cover, and who holds a node with real v0.2.5 or v0.2.7 data to run it on?
- Does a production node answer DelegateError::Missing for a delegate whose WASM was unregistered while its secrets remain, and should the UI register the frozen WASM to reach them? It never does (ui/src/migration/legacy_delegates.rs:73-74), and a release build does not embed that WASM.
- Do v0.2.5 delegate replies match the wire-v1 types exactly? Only bytes from the 206-era delegates are captured.
- Which FailureCode does the mail delegate give for a byte-budget refusal, and can Refused on an owned write mean anything other than 'identity being deleted'? Both decide permanence in successor.rs:429-447; the delegate side was not read here.
- How does the crate count `attempts` for a row this module holds back by returning Err from fetch_secrets? The crate's driver (15.7k lines) was read only at its public types and docs; the hold-back tests pass in the tree's own claims but were not run.
- docs/codebase-notes/ exists inside the repository (written 2026-11-19 by an earlier step) although the task describes the fork as having no changes of its own; is that directory meant to be committed?

### Coverage

Read in full: `ui/src/migration/mod.rs`, `ui/src/migration/legacy_delegates.rs`, `ui/src/migration/successor.rs`, `ui/src/migration/scrub.rs`, `ui/src/migration/node.rs`, `ui/src/migration/ledger.rs`, `ui/src/migration/orphans.rs`, `ui/src/legacy_registry.rs`, `ui/build.rs`, `legacy_inbox.toml`, `legacy_identity_delegate.toml`, `legacy_local_state_delegate.toml`, `tools/v1-node-seed/Cargo.toml`, `tools/v1-node-seed/src/main.rs`, `.release-freeze`, `ui/tests/legacy-migration.spec.ts`

Skimmed: `ui/src/migration/tests.rs (module docs, harness lines 1-484, every test name, selected bodies)`, `ui/src/migration/tests/scrub.rs (module docs, helpers, every test name, selected bodies)`, `ui/src/migration/captured_v0_2_7.txt (header read, hex payload not decoded)`, `ui/src/app/migration_view.rs (lines 1-260)`, `ui/src/app/migration_fixture.rs (lines 1-40, 88-97)`, `ui/src/app/identity_store.rs (lines 538-585, 735-845)`, `ui/src/api.rs (hook sites near 2655-2690, 3405-3425, 3590-3602, and the test-only legacy includes)`, `ui/Cargo.toml`, `Makefile.toml (capture-v0-2-7-replies, test-e2e-legacy-migration, test tasks)`, `scripts/check-guards-fail.sh (grep of migration guards only)`, `scripts/check-release-scrub.sh (header)`, `delegates/mail-delegate/src/store.rs (lines 160-245)`, `docs/codebase-notes/README.md`, `~/.cargo/registry freenet-migrate-0.8.0: src/lib.rs, src/lineage.rs, parts of src/journal.rs and src/delegate_migrate.rs`, `~/.cargo/registry freenet-migrate-build-0.2.0: src/lib.rs, item list of src/registry.rs`

Skipped: (none)
