> Notes v1, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# ui-core-v1: UI startup, node WebSocket layer, v1 inbox client, AFT client

Paths are relative to the repo root. Tree read at upstream 2358b59, workspace version 0.2.7 (`Cargo.toml:56`). Nothing was built or run.

## 1. Purpose and position

`ui/` is one crate (`freenet-email-ui`, cdylib+rlib, bin `app`; `ui/Cargo.toml:1-13`) compiled to WASM by `dx` and served from the webapp contract. This subsystem is the part of it that:

- boots the page (`ui/src/lib.rs:121-174`) and starts the single node loop (`ui/src/app.rs:202-214` -> `api::node_comms`, `ui/src/api.rs:1734`);
- owns the only WebSocket to the node and routes every `HostResponse` (`ui/src/api.rs:2571-3732`);
- implements the v1 mail path: inbox contract client (`ui/src/inbox.rs`) and Anti-Flood Token client (`ui/src/aft.rs`);
- embeds the frozen v1 WASM (inbox, token record, token generator) and publishes them per identity.

It holds no keys. Every signature, seal and decryption is a CBOR call to the mail delegate (`ui/src/mail_delegate/`, another map), which answers over this same WebSocket. That creates the central rule of the file: the node loop must never await a delegate answer, because it is what delivers them (`ui/src/app/identity_store.rs:5-8`). Neighbours that hook into the loop: `v2::net::route` (`api.rs:2582`), `migration::node` (`api.rs:2675, 2723, 3417, 3599`), `ghostkeys` (`api.rs:2594, 3421`), `mail_delegate::deliver` (`api.rs:3435`).

## 2. Walkthrough

### ui/Cargo.toml, features
`default = ["use-node"]` (`ui/Cargo.toml:100`).

| Feature | Compiles in |
|---|---|
| `use-node` (empty, `:111`) | Real `WebApi`/`WebApiRequestClient` (`api.rs:21-29, 130-135`), `node_comms`, `contract_api`, `delegate_api`, `inbox_management`, `token_record_management`, `token_generator_management`, `identity_management`; all `include_bytes!`/`include_str!` of `published-contract/wasm/v1/*`; `mail_delegate::node` (embeds the candidate delegate, `ui/src/mail_delegate/node.rs:19-22`); `WEB_CONTAINER_CONTRACT_ID` (`lib.rs:102-104`); `build.rs` artifact presence check and `FREENET_EMAIL_FACADE_ID` (`ui/build.rs:35-107`). Initial Sent state `Pending` (`app.rs:6433-6434`). |
| `local-delegate` (`:110`) | Optional dep `mail-delegate` with `v1-transition` (`:41`), run in-page. `compile_error!` with `use-node` (`lib.rs:16-20`). |
| `no-sync` (`:108`) | Implies `local-delegate`. Selects the offline action loop (`app.rs:215-332`); `build.rs` returns early (`build.rs:37-39`). |
| `example-data` (`:104`) | Implies `local-delegate` + `test-ghost-master`. Mock identities/mail, in-memory delivery (`app.rs:45-49, 574+`), storybook (`app.rs:343-350`). |
| `test-ghost-master` (`:122`) | Test Ghost Key master and mock vault; marker logged at start (`lib.rs:161-162`). |
| `e2e-probe` (`:115`) | Only `ui/src/jobs.rs` instrumentation. Nothing in this subsystem. |

Without `use-node`: `WebApi {}` is empty, `WebApiRequestClient` is a unit struct whose `send` is a no-op (`api.rs:166-173`), the code-hash consts are `""` (`inbox.rs:59-60`, `aft.rs:29-36`), and the crate allows dead code (`lib.rs:7-10`). The valid sets are `use-node` (+`test-ghost-master`/`e2e-probe` via `FREENET_UI_FEATURES`, `Makefile.toml:526`) and `--no-default-features --features example-data,no-sync`. `freenet-migrate` is pinned `=0.8.0` (`:48`); stdlib is the workspace `0.12.1` with `net` (`:45`, root `Cargo.toml:62`); release profile is `panic = "abort"` (root `Cargo.toml:102-107`).

### ui/Dioxus.toml, index.html, freenet.toml
`Dioxus.toml`: app name `freenet-email-app`, `out_dir = "dist"`, title blank (set at runtime). `index.html`: mount `<div id="main">`, vendored same-origin CSS only (gateway CSP). `freenet.toml`: `type = "webapp"`, `source_dirs = ["dist"]`. The Makefile packs `target/dx/app/release/web/public` instead (`Makefile.toml:524, 599`), so `dist` appears unused.

### ui/src/lib.rs, bin/app.rs
`main()`: native tracing or WASM panic hook that logs `WASM PANIC: ...` (`lib.rs:138-141`); logs the embedded webapp and container ids (diagnostic only, `:147-157`); offline sets the stub `WEB_API_SENDER` (`:168-171`); `dioxus::launch(app::app)`. `bin/app.rs` calls it. `branding::app_name()` reads `ui/branding/app.json` (`"Mail"`).

### ui/src/api.rs (4021 lines)
- **Connection** (`:45-114`): URL is `{ws|wss}://{location.host}/v1/contract/command?encodingProtocol=native` (`:75`). stdlib `WebApi::start` with a result handler that forwards into an unbounded channel, an error handler that only logs `host error:` (`:98-100`), and an `onopen` oneshot. `WEB_API_SENDER: OnceLock<WebApiRequestClient>` (`:18`) is the global handle every other module clones.
- **`contract_api::create_contract`** (`:478-502`): PUT with `subscribe: true` (#288).
- **`delegate_api`** (`:546-618`): `RegisterDelegate` with the pinned stdlib 0.3.5 `V0_2_CIPHER`/`V0_2_NONCE` (`:570-578`), also used by the mail delegate registration.
- **`contact_import`** (`:687-756`): `PENDING` (instance id -> typed address) and `RESULTS` thread-locals polled by the import modal; `decode_import_fetch` (`:625-677`) takes vk from contract params, ek and policy from state, and the inbox code hash from the response key.
- **`inbox_management`** (`:759-1026`): `INBOX_CODE` embed (`:767-768`); `CREATED_INBOX`, `AWAITING_MIGRATION_ACK`, `PENDING_MIGRATION`, `MIGRATED_IDENTITIES`; `select_owner_ek` (`:843-851`); `put_migrated_inbox` (`:853-920`, delegate re-signs, refuses a diverged ek at `:896-900`); `migrate_inbox` (`:928-972`, one GET per candidate hash, no subscribe).
- **`token_record_management`** (`:1029-1169`): `TOKEN_RECORD_CODE` embed; mirror statics; `create_contract` PUTs `TokenAllocationRecord::default()`; `put_migrated_aft_record` re-PUTs unchanged JSON.
- **`token_generator_management`** (`:1172-1289`): embeds `freenet_token_generator` + code hash; `ensure_registered` (`:1226-1267`) registers once per page per delegate key and waits up to `REGISTRATION_WAIT_MS = 15_000` (`:1213`) for an empty-values `DelegateResponse`; on timeout it logs and returns `Ok(())` without marking registered.
- **`identity_management`** (`:1296-1405`): `PENDING_CONFIRMATION` keyed by encoded vk, `NewIdentity.created()` needs identity + inbox ack + AFT-record ack + generator ack; then `alias_creation` (`:1341-1378`) routes the keys and issues GET-with-subscribe for both contracts.
- **Inbox state plumbing**: `set_inbox_state` (`:1435-1473`), `SeenStates` (`:1515-1563`, decides when stale read-marks may be reclaimed), `SELF_HEAL_PUTS` (`:1584`), `spawn_decrypt` (`:1669-1730`, `DECRYPT_RETRIES = 3`, backoff `2000 << attempt` ms).
- **`node_comms`** (`:1734-3854`): see section 3.
- `TryNodeAction` (`:3879-3888`), `node_response_error_handling` (`:3858-3876`) which pushes `Err((error, action))` into the loop's `client_errors` channel.

### ui/src/inbox.rs (2610 lines)
- Hashes and keys: `INBOX_CODE_HASH` (`:56-60`), `LEGACY_INBOX_CODE_HASHES` = one entry `Ht8rc...` (`:71-76`), `build_migration_candidates` (`:88-100`), `select_migrate_from` (`:118-128`), `inbox_key_for` / `inbox_key_for_with_hash` (`:145-169`), `verify_message_signature` (`:175-191`).
- Thread-locals (`:210-232`): `PENDING_INBOXES_UPDATE` (inbox -> sealed messages awaiting a token), `INBOX_TO_ID` (own inbox key -> `Identity`, the routing table), `PENDING_SENT_ACK` (inbox -> FIFO of `PendingSentEntry`).
- Sent tracking: `PENDING_SENT_TIMEOUT_SECS = 90` (`:254`), `enqueue_pending_sent_ack` (`:278-295`), `fail_stranded_pending_sent` (`:323-348`), `take_next_pending_sent_ack` (`:351-361`), `fail_pending_sent_for_inbox` (`:370-384`), `expire_stale_pending_sent` (`:455-469`).
- `MessageModel::finish_sending` (`:621-693`), `DecryptedMessage` (`:721-745`, JSON plaintext; threading fields `#[serde(default)]`), `start_sending` (`:749-858`), `finish_sending_bypass` (`:908-938`), `sign_authority` (`:943-954`).
- `ml_kem_encrypt`/`ml_kem_decrypt`/`hkdf_derive_key` are `#[cfg(test)]` mirrors only (`:960-1036`): `[24-byte nonce | 1088-byte ML-KEM-768 ct | XChaCha20-Poly1305 ct]`, key = HKDF-SHA256(ss, info `freenet-email-v1 message-key`). Production crypto is in the delegate.
- `InboxModel` (`:1040-1057`): `messages`, private `settings`, `key`, and four hash sets `removed`/`known`/`in_flight`/`unreadable`. `from_state`, `take_new`, `release`, `retake`, `replace_state`, `apply_delta`, `add_decrypted` (`:1337-1496`); `remove_messages` (`:1256-1305`, batches of `MAX_REMOVAL_BATCH = 1024`, `protocol/src/lib.rs:34`); `settings_modify_prepare` (`:1560-1580`); `get_state` = GET `subscribe: true` (`:1582-1600`).
- `inbox/crypto_vectors.rs`, `kem_fixed_rng.rs`: known-answer tests (section 6).

### ui/src/aft.rs (1294 lines)
- `TOKEN_RECORD_CODE_HASH`, `TOKEN_GENERATOR_DELEGATE_CODE_HASH` (`:26-36`); `LEGACY_TOKEN_RECORD_CODE_HASHES = &[]` (`:57`).
- Thread-locals (`:68-77`): `RECORDS` (`Identity` -> `TokenAllocationRecord`), `PENDING_TOKEN_ASSIGNMENT` (delegate -> inboxes), `PENDING_CONFIRMED_ASSIGNMENTS` (record id -> registers), `PENDING_INBOXES_UPDATES` (`(inbox, assignment_hash)`), `VALID_TOKEN` (never used).
- Token-call accounting for attributing keyless delegate errors: `TOKEN_CALLS`, `TOKEN_CALL_WINDOW_MS = 60_000` (`:79-150`).
- `PENDING_ASSIGNMENT_TIMEOUT_SECS = 30`, `PENDING_ASSIGNMENT_SWEEP_INTERVAL_MS = 5_000` (`:158, 165`).
- `pick_spend_tier` (`:184-186`), `build_recipient_criteria` (`:193-204`), `record_key_for` (`:212-223`), `load_all` (`:225-249`), `expire_stale` (`:281-311`), `drain_pending_for_delegate` (`:320-339`), `confirm_allocation` (`:348-398`), `allocated_assignment` (`:400-454`), `assign_token` (`:463-589`), `set_identity_contract` (`:591-611`), `record_state_for` (`:625-631`), `seed_default_record` (`:640-646`), `update_record` (`:648-672`), toast text (`:725-811`).

### Small files
- `rng.rs`: `random_bytes<N>()` over `getrandom::fill` (`:9-13`); `crypto_rng()` test-only.
- `log.rs`: `info` (console + tracing), `debug!` compiled out without `debug_assertions` (`:24-32`), `error(msg, Option<TryNodeAction>)`, `local_state_failure` (log + toast, names `CapReached`).
- `pending_saves.rs`: a counter guard; publishes `data-pending-saves` on `<html>` and installs a `beforeunload` warning while > 0 (`:52-78`).
- `clipboard.rs`: `copy_text` tries `execCommand`, Clipboard API, then the gateway shell `postMessage` proxy addressed to the gateway origin, never `"*"` (`:179-183`).
- `toast.rs`: `Signal<Vec<Toast>>`; TTL 2.2 s info/success, 8 s warning/error (`:68-73`); `push_toast` falls back to `ROOT_QUEUE` outside a component (`:119-139`); `push_toast_detached` for detached tasks.
- `test_util.rs`: `test_assignment`, `frozen_key` (reads `ui/src/frozen_keys_v1.txt`), `frozen_fixture_vk` (seed `[7;32]`).

## 3. Interfaces

### Startup sequence
1. `app()` provides signals, then `use_coroutine` runs `node_comms` through `future_to_promise` and stores the handle with `identity_store::set_action_sender` (`app.rs:202-214`).
2. `node_comms`: `WebApi::new()`, await socket open (`api.rs:1765-1771`); `load_all` over `user.identities` (empty on a fresh page, `:1773-1778`); `WEB_API_SENDER.set` (`:1783`); spawn `identity_store::load_and_report()` (`:1788`) and the contacts load with 3 retries (`:1793-1882`).
3. `load_identities` (`identity_store.rs:813-900`): list from the mail delegate, start v2 owner, `prepare` each identity (writes migration markers first, `:973-1004`), send `NodeAction::IdentitiesLoaded`, then load local state and call `fail_stranded_pending_sent` (`:878`).
4. `identities_loaded` (`api.rs:1889-1954`): upsert identities, `InboxModel::load_all` and `AftRecords::load_all` (routing entry first, then GET `subscribe: true`), prime every contact inbox, dispatch migration GETs.
5. Loop (`api.rs:3738-3853`): `select!` over host responses, `NodeAction`s, outgoing requests (`api.api.send(req).await.unwrap()`, `:3768`), `client_errors`, and a 5 s sweep.

### Requests sent
GET (own inbox/AFT record with `subscribe: true`; contact prime with `return_contract_code: true, subscribe: true`, `api.rs:432-437`; import fetch and migration GETs without subscribe), PUT (`subscribe: true`), UPDATE (`Delta` JSON, or `RelatedStateAndDelta`), `DelegateOp::RegisterDelegate`, `DelegateOp::ApplicationMessages`. There is no standalone Subscribe, no unsubscribe, no reconnect.

### Responses handled (`handle_response`)
- v2 first (`:2582`). Errors: ghostkeys `DelegateError::Missing` (`:2592-2594`); `ContractError::Update` (`:2595-2632`); `ContractError::Get` resolves a pending import (`:2633-2668`); other contract errors logged as `FIXME` (`:2669-2671`); `DelegateError` attributed to migration, mail delegate or token call (`:2672-2696`); `RequestError::Disconnect => todo!()` (`:2697-2699`).
- `GetResponse` precedence (`:2713-3023`): own inbox (`INBOX_TO_ID`), own AFT record (`token_rec_to_id`), import `PENDING`, contact inbox (feeds `contact_tier_cache`), `PENDING_MIGRATION`, `PENDING_AFT_MIGRATION`, else "unknown contract key".
- `UpdateNotification` (`:3024-3231`): own inbox `Delta` -> `apply_delta` + decrypt + one refetch; `State` -> `set_inbox_state(Pushed)`; any other variant `unreachable!()` (`:3118`); AFT record -> `update_record`; contact inbox -> tier cache; own-AFT code-hash backstop.
- `UpdateResponse` (`:3232-3308`): pops the next pending Sent ack for that key, then, if the key is an own AFT record, parses the summary (or a whole record) and calls `confirm_allocation`.
- `PutResponse` (`:3309-3415`): self-heal ack, new-identity acks, migration acks (`identity_store::migration_landed`).
- `DelegateResponse` by key (`:3416-3591`): migration, ghostkeys, mail delegate (`deliver`, or registration ack), else the token generator: empty values = registration answer; payload = bincode `TokenDelegateMessage` (`AllocatedToken`, `Failure`, `RequestNewToken` echo ignored).
- `Ok`, `SubscribeResponse` ignored; `NotFound` classified by `classify_notfound` (`:3598-3727`); anything else "message not handled".

### Wire formats
- Inbox state/deltas: `serde_json` of `freenet_mail_wire_v1::inbox::{Inbox, UpdateInbox, Message, InboxSettings, InboxParams}` (`wire-v1/src/inbox.rs:37-198`). Authority payload `STATE_UPDATE || ek || json(settings) || micros BE`.
- AFT: record/summary/params JSON; delegate messages bincode 1 (`wire-v1/src/aft.rs:39-51`); token record delta is one JSON `TokenAssignment` (`aft.rs:434-437`).
- Mail delegate ops used here: `SealV1Message`, `DecryptV1` (batches of 32), `SignV1Authority`, `SignV1Removal`, `V1AftParams` (`ui/src/mail_delegate/ops.rs:351-543`), 30 s timeout each (`ui/src/mail_delegate/mod.rs:81-90`).
- Storage: none in the browser. Sent rows and migration markers are mail-delegate items (`ItemKind::Sent`, `IdentitySettings["migration"]`, `identity_store.rs:603-611`).

### v1 send, end to end
1. `send_now` (`app.rs:6164`): confirmed-identity gate, v2 recipients diverted (`:6186-6240`), `resolve_for_send`, derive the recipient inbox key under the contact's `inbox_wasm_hash` (`:6305-6311`), take tier from `contact_tier_cache` else the card (`:6322-6342`).
2. `InboxView::send_message` (`app.rs:479-568`) builds the future; it is `spawn_forever`'d with a `pending_saves` guard (`:6399-6411`). Synchronously: `local_save_sent` as `Pending`, `enqueue_pending_sent_ack(inbox_key, alias, sent_id, sent_at_ms)` (`:6453-6466`), then `save_sent` to the delegate (`:6498-6505`).
3. `start_sending` (`inbox.rs:749`): `seal_v1` in the delegate; `hash = blake3(ciphertext)`; if the cached recipient policy allows the verified bypass, `finish_sending_bypass` sends a bare `Delta` with a placeholder assignment (`:791-798, 908-938`).
4. Else `assign_token` (`aft.rs:463`): `V1AftParams` (the signing seed), `ensure_registered`, `RECORDS` lookup (miss = error with `NO_TOKEN_RECORD_MSG_PREFIX`), `pick_spend_tier`, send `RequestNewToken { request_id, criteria, records, assignment_hash }` with `request_id` from a per-page `AtomicU32` starting at 0 (`:472, 567`), push `(inbox, hash)` to `PENDING_INBOXES_UPDATES`. Back in `start_sending`: `pending_assignment`, queue `PendingOutgoing` (`inbox.rs:845-855`).
5. The token generator asks the user through the host's permission overlay on every request (`modules/antiflood-tokens/delegates/token-generator/src/lib.rs:162-193`); the page has no prompt UI.
6. `AllocatedToken` (`api.rs:3510-3544`) -> `allocated_assignment`: UPDATE the own AFT record with the assignment, start the 30 s register.
7. `UpdateResponse` for the AFT record -> `confirm_allocation` appends to `RECORDS` and calls `finish_sending`, which pops the first queued message for that inbox and sends `RelatedStateAndDelta { related_to: token_record, state: RECORDS json, delta: AddMessages }` (`inbox.rs:666-688`).
8. `UpdateResponse` for the recipient inbox -> row `Delivered` (`api.rs:3236-3263`).

### v1 receive, end to end
Own inbox GET-with-subscribe at load -> `GetResponse` -> `set_inbox_state(Fetched)` -> `InboxModel::from_state`/`replace_state` returns `PendingDecrypt` -> `spawn_decrypt` -> `decrypt_v1` -> `add_decrypted` verifies the sender signature and pushes `MessageModel { id = message_id_for(ciphertext) }` (`inbox.rs:1483-1494`). Live mail arrives as `UpdateNotification`. An own inbox `NotFound` triggers the #288 self-heal re-PUT (`api.rs:3653-3687`) unless the identity is unconfirmed. Separately the mail delegate's background v1 watch imports arrivals as kept copies (AGENTS.md "v1 watch"); that path does not pass through this code.

### How a pending send is tracked
Everything between the Send click and the inbox UPDATE lives in page memory: the sealed ciphertext (`PENDING_INBOXES_UPDATE`), the hash queue, the token register, and `PENDING_SENT_ACK`. Only the Sent row (`Pending`) is persisted. Resolution inside a live page: `Delivered` on the inbox `UpdateResponse`; `Failed` on `start_sending` error (`app.rs:544-565`), delegate `Failure` (`api.rs:3545-3569`), AFT confirmation older than 30 s (`:3800-3828`), or no answer 90 s after the click (`:3836-3850`). If the tab closes or reloads before the UPDATE is dispatched the message is lost; there is no resend. The next load marks `Pending` rows that are at least 90 s old as presumed failed (`inbox.rs:323-348`, `local_state.rs:847-869`); a later `Delivered` may still overwrite a presumed failure (`ui/src/local_state/items.rs:446-456`).

## 4. Invariants

- Node loop never awaits the mail delegate; signing work is `spawn_detached` (`api.rs:1411-1429, 2147-2171, 2876-2913, 3666-3686`). Convention only, no test.
- Embedded WASM matches its embedded code hash: `embedded_*_has_the_embedded_code_hash` (`api.rs:771, 1040, 1185, 1327`) via `embedded_artifact` (`:509-538`). The v1 hash files have no trailing newline and are used untrimmed (`api.rs:217`, `inbox.rs:168`).
- Key derivation equals the v0.2.x client: `inbox_key_matches_the_v0_2_client` (`inbox.rs:2463`), `record_key_matches_the_v0_2_client` (`aft.rs:899`), `delegate_key_matches_the_v0_2_client` (`api.rs:1194, 1332`), against `ui/src/frozen_keys_v1.txt`.
- Register cipher/nonce are stdlib 0.3.5's (`api.rs:589-617`, `ui/src/register_defaults_v1.txt`).
- Legacy lists are append-only and exclude the current hash (`inbox.rs:1698`, `aft.rs:887`).
- Contact inbox keys go through `inbox_key_for_with_hash` with the contact's hash; bs58 without lowercasing (`inbox.rs:2439, 2475`).
- Terminal Sent states never flip (`local_state.rs:811-833`).
- Migration marker cleared only on the PUT ack (`api.rs:3385-3414`; `awaiting_ack_only_clears_on_matching_key_260`).
- An unconfirmed identity publishes nothing (`classify_notfound`, source-text tests `api.rs:327-402`).
- Crypto dependency bumps reproduce v1 vectors (`inbox/crypto_vectors.rs`).
- A decryption never resurrects a removed or superseded message (`inbox.rs:2077-2200`).

## 5. State

- Frozen and shipped: `published-contract/wasm/v1/{freenet_email_inbox, freenet_token_allocation_record, freenet_token_generator}` and their code hashes (inbox `Av3KF7...`, record `4cLK44...`, generator `H4kgUi...`).
- Candidate: the mail delegate this code depends on is `published-contract/candidate/mail_delegate`, built with `v1-transition`.
- Transitional: the whole AFT send path. `V1AftParams` exists only under `v1-transition` (`delegates/mail-delegate/src/handlers.rs:1584-1600`); `scripts/check-release-delegate.sh` refuses a release >= 0.3.0 whose delegate has it or whose UI references `published-contract/candidate`. As written, `assign_token` (`aft.rs:483-485`) and `identity_store::publish` (`identity_store.rs:1122-1126`) fail against a release delegate.
- Unfinished: reconnect; several placeholders (section 10).

## 6. Tests

All are `#[cfg(test)]` unit tests inside the owned files.
- `api.rs`: `notfound_routing` (6 tests, two read source text), `delegate_api` (2), embedded-artifact (4), `select_owner_ek` (3), `seen_states_tests` (4), `awaiting_ack_tests` (1). No test drives `handle_response` or `node_comms`.
- `inbox.rs`: `migration_logic_tests` (candidate chain, `select_migrate_from` table, serde back-compat); `tests` (removal, signature verification, #267 undecryptable messages, superseded/in-flight/retry decrypt logic, removal batching, settings deltas, key derivation, stale-send sweep, #286 pending count). `crypto_vectors` (9 known-answer tests).
- `aft.rs`: token-call window (2), legacy/frozen keys, criteria, `drain_pending_for_delegate` (2), toast text, `pick_spend_tier` (5). No test covers `assign_token`, `allocated_assignment`, `confirm_allocation`, `update_record` or `expire_stale`.
- `toast.rs` (11), `log.rs` (3), `pending_saves.rs` (1).

Commands: `cargo test -p freenet-email-ui --lib` (native, default `use-node`; runs the `use-node`-gated tests) and `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` (`cargo make test-ui-offline`; runs the `local-delegate`-gated tests that sign and decrypt through the real delegate, `inbox.rs:2133, 2236`). `cargo make test` runs both plus excluded crates. `cargo make check-golden-capture` regenerates `kem_vectors_v1.txt` and `register_defaults_v1.txt` through the frozen crates and diffs. End to end: `cargo make test-e2e-real-node` (`ui/tests/live-node.spec.ts`; needs `freenet`, `fdev`, `dx`, Playwright browsers) covers cross-node send/receive, self-heal, tier change.

## 7. Changing it safely

- Any change to `protocol/` or `wire-v1/` types used here forces a candidate delegate rebuild (`scripts/check-delegate-candidate.sh`) and may hit `protocol/src/golden_v1.txt` / `wire-v1/src/golden_v1.txt`.
- Never change the JSON shape of `DecryptedMessage` incompatibly: old ciphertexts must still parse; new fields need `#[serde(default)]`.
- A new inbox or AFT generation: add `published-contract/wasm/v<N>/`, repoint the `include_*!`s (`api.rs:767, 1036, 1179-1182`, `inbox.rs:57`, `aft.rs:27-34`), append the old hash to the in-source slice and to `legacy_inbox.toml`, update `ui/src/legacy_registry.rs` pins and `build.rs`'s required list (`build.rs:52-74`).
- Thread-local state is per page and unsynchronised across tabs; tests that touch it clear it by hand (`inbox.rs:2522`, `aft.rs:826, 993`).
- Toasts from the loop: `push_toast` works only by its root-queue fallback; prefer `push_toast_detached`.
- Any panic inside `node_comms` ends response handling and the sweeps while the UI keeps rendering (section 9).
- Both `clippy` configurations matter: offline builds compile different code.

## 8. Docs and comments versus code

1. AGENTS.md gives the prime GETs at `api.rs` "~1442 and ~2540"; they are `prime_contact_inbox` (`api.rs:420-470`) and the `CreateContact` arm (`:2271-2313`).
2. AGENTS.md says to append a bumped hash to the slice and to `legacy_inbox.toml`. The TOML has two rows (`Ht8rc...`, `9Kuhod...`), the slice one (`inbox.rs:71-76`). `legacy_registry.rs:8-9` admits the TOML list is not read at runtime.
3. `api.rs:2607-2616` says a failed UPDATE to a recipient inbox flips its Sent rows. The guard is `InboxModel::contract_identity(key)`, which only knows own inboxes (`inbox.rs:212, 1244`).
4. `api.rs:924-927` says the caller stamps the hash "so subsequent reloads don't retry forever"; the pending marker is kept until a PUT ack, so a vanished old inbox is retried every load. `:794-795` mentions a timeout that does not exist.
5. `api.rs:812-817, 1105-1110` say migration reuses `Inbox::new` over a fixed salt; it uses `sign_authority` + `Inbox::with_signature` over the full payload.
6. `api.rs:3444-3449, 3489-3494` describe an identities-delegate response; the arm is the token-generator catch-all.
7. `inbox.rs:50-55`, `aft.rs:23-25` say the hashes come from `cargo make build-*`; they are committed frozen files.
8. `inbox.rs:238-245` budgets 30 s for "minting"; the 30 s timer starts only after the token arrives (`aft.rs:441-445`), and the user's prompt time counts against the 90 s. `aft.rs:271-272` calls the sweep "60s".
9. README.md:44-45 "first send burns one token, minted automatically": every non-bypass send requests one and the delegate prompts. README.md:77-80 names `modules/identity-management` as key holder; it is not run (`api.rs:1308-1310`). README.md:64 says ChaCha20-Poly1305; the envelope is XChaCha20-Poly1305.
10. `lib.rs:108-111` says the offline build "skips build.rs"; it runs and returns early.

## 9. Suspected bugs and weak spots

- **No reconnect, and a dropped socket kills the loop** (high): stdlib `send` returns `Err` when the socket is not open; `api.rs:3768` unwraps; `Disconnect` is `todo!()` (`:2698`); `onclose` only logs. After the panic the page still renders, but no response is processed and no sweep runs, so in-flight sends stay `Pending` until reload.
- **Reload within 90 s leaves a row `Pending` for the whole next session** (high): `fail_stranded_pending_sent` runs only inside `load_identities` and skips rows younger than 90 s; nothing re-checks.
- **Failed UPDATE to a contact's inbox is silently dropped** (high): item 3 above; no log, no toast, row waits for the 90 s sweep.
- **False `Failed` then delivery** (medium-high): the 90 s sweep does not clear `PENDING_INBOXES_UPDATE` (`inbox.rs:455-469` versus `:370-384`), so a token approved late still delivers; the row stays `Failed`, inviting a duplicate.
- **Custom relay URL can never apply** (high): `WebApi::new` reads `global_settings()` (`api.rs:71`) before any state is loaded; state lives behind the socket being opened.
- **Self-heal and restore reset inbox policy** (medium): `initial_inbox_state` uses `Identity.required_tier`, always `Min10` (`login.rs:93`), default settings and a fresh `last_update`; contract merge is last-write-wins on settings (`contracts/inbox/src/lib.rs:623-632`).
- **`update_record` replaces `RECORDS` with a `Delta`** (medium-low, depends on core): the contract's delta holds only missing assignments (`modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs:160-185`); a shrunken record makes the generator reuse a taken slot.
- **`request_id` restarts at 0 per page** (medium): two tabs collide on the delegate's `pending_request_<id>` secret.
- **`unreachable!()`** at `api.rs:3118` and `:3541` (low): any other `UpdateData` variant or an unrouted record kills the loop.
- **Leaks** (low): `PENDING_INBOXES_UPDATES` and `PENDING_TOKEN_ASSIGNMENT` are never cleared on success, so a later delegate `Failure` fails every inbox ever sent to.
- **New-identity race** (low): if all three acks precede the `CreateIdentity` action, `alias_creation` never runs and the AFT record is not loaded until reload.
- **FIFO pairing** in `finish_sending` and `take_next_pending_sent_ack` (low): two sends to one inbox can swap tokens or acks; the contract does not check hash against content.

## 10. Open work

`todo!` reconnect (`api.rs:2698`); `FIXME` error handling (`:2590, 2597, 2670`); "don't unwrap" (`:1741`); retry on slot collision (`:3530`); retry after unconfirmed inbox update (`inbox.rs:689-690`); propagate missing record to UI (`aft.rs:516`); placeholders `StoredDecryptedSettings`, `InboxSummary` (`inbox.rs:545-558`), `MAIN_ELEMENT_ID` (`lib.rs:46-47`), `VALID_TOKEN` (`aft.rs:75`); recipient hash never refreshed after import (AGENTS.md, #251 improvement 1); v1 send path removal before 0.3.0.

## 11. Unanswered

- Does core deliver client `UpdateNotification`s as `State`, a contract delta, or the applied `UpdateData`? Decides two items in section 9.
- Does core answer `RegisterDelegate` with an empty-values `DelegateResponse` in every supported version (`ensure_registered` waits 15 s per send otherwise)?
- Is a late `AllocatedToken` delivered to a new page after reload?
- What replaces `V1AftParams` at 0.3.0: is v1 sending dropped entirely?
- Does `cargo make dev` (`dx serve --features use-node`) reach a node, given the URL comes from `location.host`?
- Is `ui/freenet.toml` still consumed by anything?
- The text of upstream issue #320 was not read; the stuck-`Pending` causes above are from code only.

---

## Appendix: the reader's structured summary

This subsystem boots the Dioxus page, owns the single WebSocket to the node, and implements v1 mail: the inbox contract client (ui/src/inbox.rs) and the Anti-Flood Token client (ui/src/aft.rs). One loop, api::node_comms, sends every request and routes every HostResponse; the page holds no keys, so sealing, decrypting and signing are calls to the mail delegate that return over the same socket, and the loop must never await one.

It embeds the frozen v1 inbox, token-record and token-generator WASM and publishes them per identity. Receiving is a GET-with-subscribe on the own inbox, then decryption in the delegate. Sending is a five-hop in-memory state machine: seal, fetch the signing seed (V1AftParams), ask the token-generator delegate (which prompts the user on every send), UPDATE the own token record, then UPDATE the recipient inbox.

Current state: it works on the happy path and is covered by unit tests on its pure helpers plus a real-node Playwright suite, but nothing tests the loop itself. There is no reconnect; a dropped socket panics the loop. A pending send lives only in page memory: closing or reloading the tab loses it, and only the Sent row persists. The whole AFT send path depends on a delegate feature (v1-transition) that the 0.3.0 release check refuses, so it cannot ship as written. I found several concrete causes of rows stuck in Pending or wrongly Failed; none was confirmed by running anything, and I did not read the text of issue #320.

### Key facts

- Feature matrix: default is use-node; example-data implies local-delegate + test-ghost-master, no-sync implies local-delegate, and local-delegate with use-node is a compile_error, so the only valid sets are use-node (optionally + test-ghost-master / e2e-probe via FREENET_UI_FEATURES) and --no-default-features --features example-data,no-sync. ui/Cargo.toml:99-122, ui/src/lib.rs:16-20, Makefile.toml:526
- Without use-node the WebSocket client is a unit struct whose send is a no-op and all v1 code-hash consts are empty strings; the crate allows dead code in that mode. ui/src/api.rs:166-173, ui/src/inbox.rs:56-60, ui/src/aft.rs:26-36, ui/src/lib.rs:7-10
- Frozen v1 WASM is embedded in api.rs only: inbox at ui/src/api.rs:767-768, token record at :1036-1037, token generator and its hash at :1179-1182; code hashes at ui/src/inbox.rs:57-58 and ui/src/aft.rs:27-34. The v1 hash files have no trailing newline and are used untrimmed. ui/build.rs:52-74 fails the build if any is missing.
- The mail delegate this code calls is still the candidate, built with v1-transition: ui/src/mail_delegate/node.rs:19-22. scripts/check-release-delegate.sh:72-86 refuses a release >= 0.3.0 that references published-contract/candidate or carries the v1-transition marker.
- The v1 AFT send path and new-identity publish both need V1AftParams (the identity's signing seed), which a release delegate refuses with 'this build has no v1 send path': ui/src/aft.rs:483-485, ui/src/app/identity_store.rs:1122-1126, delegates/mail-delegate/src/handlers.rs:1584-1600. Workspace version is 0.2.7 (Cargo.toml:56).
- WebSocket URL is derived from location.host: {ws|wss}://host/v1/contract/command?encodingProtocol=native (ui/src/api.rs:75). WEB_API_SENDER (ui/src/api.rs:18) is the global request handle, set only after the socket opens (:1771, :1783).
- The node loop is one select! over host responses, NodeActions, outgoing requests, client errors and a 5 s sweep: ui/src/api.rs:3738-3853. It is started through future_to_promise from ui/src/app.rs:202-214.
- Rule: the node loop must never await a mail-delegate answer because it delivers them; signing work is spawned detached (ui/src/app/identity_store.rs:5-8, ui/src/api.rs:1411-1429). Enforced by convention only, no test.
- All subscriptions are GET or PUT with subscribe: true; there is no standalone Subscribe, no unsubscribe and no reconnect. ui/src/inbox.rs:1592-1597, ui/src/aft.rs:708-713, ui/src/api.rs:493-499, :432-437
- Response routing precedence for GetResponse: own inbox (INBOX_TO_ID), own AFT record (token_rec_to_id, a local of node_comms), import PENDING, contact inbox, inbox migration, AFT migration. ui/src/api.rs:2713-3023. DelegateResponse is routed by key: migration, ghostkeys, mail delegate, else token generator (:3416-3591).
- v1 send state is five page-memory thread-locals: PENDING_SENT_ACK and PENDING_INBOXES_UPDATE (ui/src/inbox.rs:210-232) and PENDING_TOKEN_ASSIGNMENT, PENDING_CONFIRMED_ASSIGNMENTS, PENDING_INBOXES_UPDATES (ui/src/aft.rs:68-77). Only the Sent row (Pending) is persisted, in the mail delegate (ui/src/app.rs:6453, :6498-6505).
- Timeouts: AFT confirmation 30 s counted from token arrival (ui/src/aft.rs:158, :441-445); Sent row 90 s counted from the Send click (ui/src/inbox.rs:254); sweep every 5 s (ui/src/aft.rs:165); token-generator registration wait 15 s (ui/src/api.rs:1213); each mail-delegate call 30 s (ui/src/mail_delegate/mod.rs:81-90).
- A send is lost if the tab closes or reloads before the inbox UPDATE is dispatched; there is no resend. On the next load, Pending rows at least 90 s old are stored as presumed failed, which a late Delivered may overwrite. ui/src/inbox.rs:323-348, ui/src/local_state.rs:847-869, ui/src/local_state/items.rs:446-456, ui/src/app/identity_store.rs:878
- The token-generator delegate asks the user for permission on every token request; the page has no prompt UI and only ignores the echoed request. modules/antiflood-tokens/delegates/token-generator/src/lib.rs:162-193, ui/src/api.rs:3570-3574
- The inbox UPDATE bundles the sender's cached token record inline as RelatedStateAndDelta; the verified-sender bypass sends a bare Delta with a placeholder assignment. ui/src/inbox.rs:666-688, :908-938
- Receive: inbox state is JSON; messages are handed to the delegate in batches of 32 and added by add_decrypted, which drops anything removed or superseded meanwhile; message id is derived from the ciphertext hash. ui/src/api.rs:1435-1473, :1669-1730, ui/src/inbox.rs:1360-1496
- Inbox migration: the in-source legacy list has one hash, the AFT list is empty; candidates are GET in turn and the first response wins; the pending marker is cleared only when the re-PUT is acknowledged. ui/src/inbox.rs:71-128, ui/src/aft.rs:57, ui/src/api.rs:928-972, :3385-3414, ui/src/app/identity_store.rs:949-1040
- Tests pin key derivation against the v0.2.x client, the embedded WASM against its hash, the stdlib 0.3.5 register cipher/nonce, and v1 crypto vectors; nothing tests handle_response, node_comms, assign_token, allocated_assignment, confirm_allocation or update_record. ui/src/api.rs:509-538, :589-617, ui/src/inbox.rs:2463, ui/src/aft.rs:899, ui/src/inbox/crypto_vectors.rs:35-122
- Two unit-test configurations are needed: cargo test -p freenet-email-ui --lib (use-node gated tests) and the same with --no-default-features --features example-data,no-sync (tests that sign and decrypt through the in-page delegate). Makefile.toml:908-919, ui/src/inbox.rs:2133, :2236
- Release profile is panic = abort (Cargo.toml:102-107) and the WASM panic hook only logs (ui/src/lib.rs:138-141); node_comms contains unwrap, todo! and unreachable! on live paths (ui/src/api.rs:2698, :3118, :3541, :3768).

### Docs or comments that disagree with the code

- **[high]** Claim (code comment, ui/src/api.rs:2607-2616): A failed UPDATE to a recipient inbox is the terminal signal that a send did not land, and flips every Sent row queued against that inbox to Failed with a toast. **Reality:** The branch is guarded by InboxModel::contract_identity(key), which looks up INBOX_TO_ID, a table of the user's own inboxes only. For a contact's inbox the error matches no branch: no log, no toast, no state change. The row waits for the 90 s sweep. Evidence: ui/src/api.rs:2595-2632; ui/src/inbox.rs:212, 1244-1254
- **[medium]** Claim (AGENTS.md, 'Chained migration (#251 improvement 2)'): On a deliberate inbox bump, append the previous hash to the LEGACY_INBOX_CODE_HASHES slice and to legacy_inbox.toml. **Reality:** The two lists already differ: legacy_inbox.toml has two rows (Ht8rc..., and 9Kuhod... for the live v0.2.5 webapp), the in-source slice that the runtime walks has only Ht8rc.... legacy_registry.rs says the TOML inbox list is not read at runtime. Evidence: ui/src/inbox.rs:71-76; legacy_inbox.toml (two [[contract]] rows); ui/src/legacy_registry.rs:8-9, 67-74
- **[medium]** Claim (doc comment, ui/src/inbox.rs:238-245): The Sent-row clock is shared with an AFT phase of up to 30 s for minting, leaving the UPDATE at least 60 s. **Reality:** The 30 s timer starts only after the token has arrived. Before that the 90 s also covers seal_v1 and V1AftParams (30 s each), the 15 s registration wait and however long the user takes to answer the permission prompt. Evidence: ui/src/aft.rs:441-445; ui/src/api.rs:1213; ui/src/mail_delegate/mod.rs:81-90
- **[medium]** Claim (README.md:44-45): The first send burns one Anti-Flood Token, minted automatically by your local node. **Reality:** Every send requests a token unless the verified-sender bypass applies, and the token-generator delegate asks the user for permission on each request. Evidence: ui/src/inbox.rs:831-840; modules/antiflood-tokens/delegates/token-generator/src/lib.rs:162-193
- **[medium]** Claim (README.md:49, 77-80): The identity delegate at modules/identity-management holds the user's private keys; the system is three contracts and one delegate. **Reality:** Identities and keys live in the mail delegate. The v0.2.x identity delegate is not used at runtime; its artifacts are only pinned in a test module for migration. Evidence: ui/src/api.rs:1291-1337; ui/src/mail_delegate/node.rs:17-22
- **[low]** Claim (code comments, ui/src/api.rs:924-927 and 794-795): migrate_inbox is best-effort; on GET failure the caller stamps the current hash so subsequent reloads don't retry forever. PENDING_MIGRATION entries are removed after a timeout. **Reality:** The hash is stamped when the migration is planned, and the pending marker stays until a PUT ack. If the old inbox no longer exists, NotFound is classed Unattributed and nothing clears the marker, so the GETs are re-sent on every load. No timeout exists. Evidence: ui/src/inbox.rs:118-128; ui/src/app/identity_store.rs:676-695; ui/src/api.rs:3720-3725
- **[low]** Claim (AGENTS.md, 'Sender-side recipient hash (#251 improvement 4)'): Every helper deriving a contact inbox key uses inbox_key_for_with_hash; currently the rehydrate and CreateContact prime GETs in ui/src/api.rs (~1442 and ~2540). **Reality:** The rule holds, but the locations are stale: the prime is prime_contact_inbox at api.rs:420-470 and the CreateContact arm at 2271-2313. Lines 1442 and 2540 are unrelated code. Evidence: ui/src/api.rs:420-470, 2271-2313
- **[low]** Claim (code comments, ui/src/api.rs:812-817, 1105-1110, 2834-2838): Inbox migration reuses Inbox::new to re-sign; the signature is over a fixed STATE_UPDATE salt. **Reality:** It calls sign_authority in the mail delegate and Inbox::with_signature; the signed payload covers ek, settings and last_update. Evidence: ui/src/api.rs:901-909; ui/src/inbox.rs:943-954
- **[low]** Claim (code comments, ui/src/api.rs:3444-3449 and 3489-3494): The catch-all DelegateResponse arm handles the identities delegate (empty responses, IdentityManagement payload). **Reality:** The arm is the token-generator delegate handler: empty values is its registration answer, payloads are TokenDelegateMessage. Evidence: ui/src/api.rs:3450-3451, 3495-3575
- **[low]** Claim (code comments, ui/src/inbox.rs:50-55 and ui/src/aft.rs:23-25): inbox_code_hash is produced by cargo make build-inbox-contract; AFT hashes by cargo make build-token-*. **Reality:** They are committed frozen files under published-contract/wasm/v1/ that no build regenerates. Evidence: ui/src/inbox.rs:57-58; ui/build.rs:11-15
- **[low]** Claim (doc comment, ui/src/aft.rs:271-272): expire_stale: without the inbox key the row only failed later via the 60s send-sweep. **Reality:** The send-sweep timeout is 90 s. Evidence: ui/src/inbox.rs:254
- **[low]** Claim (README.md:64): Inbox messages are ChaCha20-Poly1305 encrypted. **Reality:** The v1 envelope is a 24-byte nonce, ML-KEM-768 ciphertext, then XChaCha20-Poly1305, keyed by HKDF-SHA256. Evidence: ui/src/inbox.rs:971-999
- **[low]** Claim (doc comment, ui/src/lib.rs:108-111): FACADE_CONTRACT_ID is empty when the offline UI build skips build.rs entirely. **Reality:** build.rs always runs (it validates the legacy registries) and returns early before setting the env var. Evidence: ui/build.rs:29-39
- **[low]** Claim (ui/Dioxus.toml:10 and ui/freenet.toml:8): Dioxus out_dir is dist and the webapp state source dir is dist. **Reality:** The build and packaging tasks use target/dx/app/release/web/public; I found nothing that reads ui/dist or ui/freenet.toml. Evidence: Makefile.toml:524, 599

### Suspected issues (leads from reading, not confirmed)

- **No reconnect: a dropped WebSocket panics the node loop** (confidence: high). stdlib's browser send returns Err when the socket is not OPEN; the loop unwraps it. RequestError::Disconnect is todo!(). onclose only logs. Any outgoing request after a drop (including the 10-minute RunJobs) ends node_comms. Inferred, not run: the page keeps rendering because other WASM entry points still work, but no response is processed and the sweeps stop, so sends already past the token request stay Pending until reload. Evidence: ui/src/api.rs:3768, 2697-2699, 98-100
- **Reload within 90 s leaves a Sent row Pending for the whole next session** (confidence: high). fail_stranded_pending_sent runs only inside load_identities and skips rows younger than PENDING_SENT_TIMEOUT_SECS. The new page has no PENDING_SENT_ACK entry for the row, so the sweep never touches it and nothing re-checks. Another open tab sees the same. Evidence: ui/src/inbox.rs:323-348; ui/src/local_state.rs:847-869; ui/src/app/identity_store.rs:878
- **A failed UPDATE to a contact's inbox is silently ignored** (confidence: high). ContractError::Update is handled only when the key is an own AFT record or an own inbox. For a real recipient nothing is logged or shown; the row is failed 90 s after the click with a generic message. Evidence: ui/src/api.rs:2595-2632
- **Row marked Failed, message delivered afterwards** (confidence: medium). The 90 s sweep removes the PENDING_SENT_ACK entry and marks the row Failed but leaves the sealed message in PENDING_INBOXES_UPDATE and its hash in PENDING_INBOXES_UPDATES. If the user answers the permission prompt later, the token arrives and the message is sent; the row stays Failed, so a resend duplicates. The late UpdateResponse can also pop a newer pending row for the same inbox. Evidence: ui/src/inbox.rs:455-469 versus 370-384; ui/src/aft.rs:409-417
- **Custom relay URL setting can never take effect** (confidence: high). WebApi::new reads global_settings() from the in-memory snapshot, which is still the default: local state is loaded from the mail delegate over the socket this function is opening, and there is no browser storage for it. Evidence: ui/src/api.rs:69-76; ui/src/local_state.rs:34, 682-684
- **Self-heal and restore re-publish the inbox with default policy** (confidence: medium). initial_inbox_state is built from Identity.required_tier and max_age_secs, which from_info always sets to Min10 and the default, with default settings (bypass off, no verified senders) and a fresh last_update. The contract merge takes settings from the newer last_update, so if any copy of the real inbox still exists its policy is overwritten. Evidence: ui/src/api.rs:3667-3675; ui/src/app/identity_store.rs:1044-1065; ui/src/app/login.rs:93-94; contracts/inbox/src/lib.rs:623-632
- **update_record replaces the cached token record with a Delta** (confidence: low). For UpdateData::Delta it parses the bytes as a whole TokenAllocationRecord and replaces RECORDS. The contract's get_state_delta returns only assignments missing from the summary. If the node notifies with such a delta, the cache loses older assignments and the generator, which fills slots from the oldest, can pick a taken slot; the record UPDATE is then rejected and the send fails after 30 s. Depends on what freenet-core sends to client subscribers, which this tree does not show. Evidence: ui/src/aft.rs:648-672; modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs:160-185
- **Token request ids restart at 0 on every page load** (confidence: medium). REQUEST_ID is a per-page AtomicU32. The token generator stashes each request under pending_request_<id> in its secret store, so two tabs sending at once overwrite each other's stash; the second approval finds no stash and returns no token. Evidence: ui/src/aft.rs:472, 567; modules/antiflood-tokens/delegates/token-generator/src/lib.rs:30-34, 66-70, 105-112
- **unreachable! on live response paths** (confidence: low). An own-inbox UpdateNotification carrying any UpdateData variant other than State or Delta, or an AllocatedToken whose record key is not in token_rec_to_id, ends the node loop. Evidence: ui/src/api.rs:3118, 3540-3542
- **Send bookkeeping is never cleared on success** (confidence: medium). PENDING_INBOXES_UPDATES entries and PENDING_TOKEN_ASSIGNMENT inboxes are removed only on expiry or delegate Failure. They grow per send, and a later Failure fails pending sends to every inbox ever sent to through that delegate. Evidence: ui/src/aft.rs:320-346, 409-417, 580-582
- **Token-generator registration timeout is swallowed** (confidence: medium). On a 15 s timeout ensure_registered logs, does not mark the delegate registered and returns Ok, so the token request goes to a possibly unregistered delegate and every later send waits 15 s again. Evidence: ui/src/api.rs:1252-1266
- **New identity can miss its token-record load** (confidence: low). alias_creation runs only from an ack handler once created() is true, which needs the CreateIdentity action to have been processed. If all three acks are handled first, or one never arrives, the AFT record is never routed or fetched and sends fail with 'token record hasn't loaded yet' until reload. Evidence: ui/src/api.rs:1341-1398, 3321-3343; ui/src/app/identity_store.rs:1128-1148
- **FIFO pairing of tokens and acks per inbox** (confidence: medium). finish_sending pops the first queued message for the inbox whichever assignment was confirmed, and UpdateResponse pops the first pending Sent row. Two sends to one recipient can swap tokens or acks. The inbox contract does not compare assignment_hash with the content, so delivery still succeeds. Evidence: ui/src/inbox.rs:626-641, 351-361; contracts/inbox/src/lib.rs:546-574

### Open work

- Reconnect after a lost node connection: todo!() at ui/src/api.rs:2698
- Error handling placeholders: FIXME at ui/src/api.rs:2590, 2597 (reject the pending assignment when the AFT record UPDATE fails), 2670; 'don't unwrap inside this function' at ui/src/api.rs:1741; 'all errors should be handled properly' at ui/src/api.rs:3863
- Retry on token slot collision: todo at ui/src/api.rs:3530
- Retry when the inbox UPDATE gets no confirmation: todo at ui/src/inbox.rs:689-690
- Propagate a missing token record to the UI or retry: todo at ui/src/aft.rs:516; avoid cloning the whole record: ui/src/aft.rs:514
- v1 send path must be removed or replaced before 0.3.0: V1AftParams is transitional (ui/src/aft.rs:474-481, ui/src/mail_delegate/ops.rs:528-537, scripts/check-release-delegate.sh:8-13)
- Seed copies decoded from V1AftParams are freed, not zeroized: ui/src/aft.rs:479-481
- Placeholders never wired: StoredDecryptedSettings and InboxSummary (ui/src/inbox.rs:545-558), MAIN_ELEMENT_ID (ui/src/lib.rs:46-47), VALID_TOKEN (ui/src/aft.rs:75)
- Recipient inbox hash is never refreshed after import; documented limit pending the in-protocol upgrade pointer (AGENTS.md '#251 improvement 1'; ui/src/inbox.rs:151-158)
- AFT record migration assumes schema-compatible bumps only; a schema change needs a reshape pass in put_migrated_aft_record: ui/src/aft.rs:46-56
- Documented open window: two tabs sending the same draft can both pass the pre-send check (ui/src/app.rs:6529-6532)
- LEGACY_TOKEN_RECORD_CODE_HASHES is empty and the legacy_inbox.toml list is 'not read at runtime yet': ui/src/aft.rs:57, ui/src/legacy_registry.rs:8-9

### Open questions

- What does freenet-core put in a client's UpdateNotification after a GET with subscribe: a whole State, the contract's delta, or the UpdateData that was applied? This decides whether update_record corrupts the token-record cache and whether the unreachable! at ui/src/api.rs:3118 can fire.
- Does every supported node version answer RegisterDelegate with a DelegateResponse that has empty values? ensure_registered and the mail-delegate registration both rely on it.
- If the page reloads while the permission prompt is open and the user then approves, is the AllocatedToken delivered to the new connection or dropped?
- What is the plan for v1 sending at 0.3.0, given the release check refuses a delegate that serves V1AftParams: is the AFT path dropped, or does the token-generator get its parameters another way?
- Does cargo make dev (dx serve --features use-node) reach a node at all? The WebSocket URL comes from location.host, which would be the dx server, and the custom relay override cannot apply at connect time.
- Is ui/freenet.toml (and Dioxus out_dir = dist) still read by any tool?
- Should the in-source LEGACY_INBOX_CODE_HASHES include 9Kuhod6Bfrd2uZAtVNtQKVWy4mku3tbfv2zop1u9HU98, which legacy_inbox.toml lists as the live v0.2.5 inbox?
- I did not read the text of upstream issue #320 (I stayed inside the local tree). The stuck-Pending causes I list come from the code alone and none was reproduced; someone should match them against the reporter's steps.

### Coverage

Read in full: `ui/Cargo.toml`, `ui/Dioxus.toml`, `ui/index.html`, `ui/freenet.toml`, `ui/src/lib.rs`, `ui/src/bin/app.rs`, `ui/src/api.rs`, `ui/src/inbox.rs`, `ui/src/inbox/crypto_vectors.rs`, `ui/src/inbox/kem_fixed_rng.rs`, `ui/src/aft.rs`, `ui/src/rng.rs`, `ui/src/log.rs`, `ui/src/pending_saves.rs`, `ui/src/clipboard.rs`, `ui/src/toast.rs`, `ui/src/test_util.rs`, `ui/build.rs (outside my paths)`, `ui/src/mail_delegate/node.rs (outside my paths)`, `ui/src/legacy_registry.rs (outside my paths)`, `legacy_inbox.toml (outside my paths)`, `scripts/check-release-delegate.sh (outside my paths)`, `README.md (outside my paths)`, `freenet-stdlib 0.12.1 src/client_api/browser.rs (cargo registry, outside the repo)`

Skimmed: `ui/src/inbox/crypto_vectors_v1.txt (line names only)`, `ui/src/inbox/kem_vectors_v1.txt (line names only)`, `ui/src/frozen_keys_v1.txt, ui/src/register_defaults_v1.txt`, `ui/src/app.rs (NodeAction, app(), send_message, the compose send handler: lines 40-360, 440-580, 6150-6550)`, `ui/src/app/identity_store.rs (lines 1-45, 540-1160)`, `ui/src/app/login.rs (Identity struct, upsert, set_alias: lines 30-260)`, `ui/src/mail_delegate/mod.rs (lines 1-330)`, `ui/src/mail_delegate/ops.rs (v1 ops and V1AftParams)`, `ui/src/local_state.rs and ui/src/local_state/items.rs (Sent delivery state functions only)`, `ui/src/contact_tier_cache.rs (lines 1-110)`, `wire-v1/src/aft.rs (lines 1-135, 350-845)`, `wire-v1/src/inbox.rs (outline only)`, `contracts/inbox/src/lib.rs (lines 300-740, 826-1015)`, `modules/antiflood-tokens/delegates/token-generator/src/lib.rs (all but tests)`, `modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs (lines 1-260)`, `delegates/mail-delegate/src/handlers.rs (v1_aft_params only)`, `Makefile.toml (build, dev, test tasks)`, `Cargo.toml (workspace)`, `published-contract/candidate/README.md (first 30 lines)`, `ui/tests/live-node.spec.ts (header and test names)`, `.github/workflows/build.yml (test commands only)`

Skipped: (none)
