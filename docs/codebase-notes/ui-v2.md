> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# ui-v2: Inbox v2 in the UI

Owned: `ui/src/v2/{mod,owner,contacts,queue,send,net,flows}.rs`, `ui/src/address.rs`, `ui/src/ghostkeys.rs`, `ui/src/app/v2_view.rs`. Refs are repo-relative `path:line`. Short names: `owner.rs` etc. mean `ui/src/v2/owner.rs`.

## 1. Purpose and position

The page-side driver of inbox v2. It signs and seals nothing: it asks the mail delegate (`Op::CreateInboxV2`, `RepublishInboxV2`, `UpdateInboxV2`, `SealV2`, `HoldPass`, `IssuePass`, `RunJobs`), moves contract states between delegate and node, and tells the user the outcome (mod.rs:1-4). AGENTS.md "Inbox v2 in the UI" describes the file split correctly.

**v1/v2 coexistence today (not in AGENTS.md):**
- v2 has **no feature gate**. `owner::start(addr)` runs for every *confirmed* identity in every build: at identity load (ui/src/app/identity_store.rs:847-851) and on each `UserInbox` render (ui/src/app.rs:3389-3395). Unconfirmed (migrated) identities publish nothing until confirmed.
- A default (`use-node`) build therefore PUTs the **candidate** contract (`published-contract/candidate/inbox_v2`, mod.rs:27-29) per identity, with params naming the production Ghost Key master (mod.rs:35-40), next to the identity's v1 inbox, which the v1 code still runs.
- **Receiving:** one list. v1 rows (live + kept) are built first, then `merge_v2_rows` replaces the v2 rows with `owner::inbound(addr, false)` (app.rs:3777-3784, 1527-1530). Stranger mail not yet accepted is held out of the list and shown in Settings (below).
- **Sending:** compose picks by the To text only (app.rs:6175-6240): an invite is refused; `contacts::recipient_addr` (a 36-char address, `name@address`, `fmail:...`, or the exact petname of a contact that has `V2Notes`) goes v2; everything else (v1 alias, own alias, 44-char bs58) goes v1 with AFT. No fallback from v2 to v1 or back. `contacts::add` refuses a 44-char legacy address: "Old contacts move over in a later update" (contacts.rs:181-187).
- **What a production user sees:** nothing yet. Releases are blocked: `.release-freeze:1-11`, and `scripts/check-release-delegate.sh:72-75` fails while anything under `ui/` references `published-contract/candidate` (mod.rs:27-29 does). A `cargo make build` of this tree would show: a new Settings screen "@ Address & new contacts" (ui/src/app/settings.rs:192,234), a "v2" note under compose's To field, and three banners above the list (app.rs:3882-3886).

## 2. Walkthrough

### mod.rs (164)
- `CODE`, `CODE_HASH` (27-29); `ghost_master()` (35-40); `params(addr)` = `inbox_v2::params_bytes(addr, master)`, master at bytes 25.. (42-44, test 146-147); `key`, `instance_id` (46-53).
- `MailBody {v, label, title, content, time_ms, thread_id, in_reply_to, quoted_excerpt}` CBOR (57-81): plaintext inside the envelope. UI-private: not in `protocol/`, no golden bytes.
- `InboxRecordView {max_version, strangers_dropped, foreign_seen}`, `LedgerView {issued_for, class, expiry, revoked}` (85-109): partial reads of the delegate's `InboxV2` and `PassLedger` items; unknown fields skipped.
- `flows` compiles only with `test + local-delegate + test-ghost-master + !use-node` (9-15).

### owner.rs (1034): the identity's own inbox
- Page state: `INBOUND`, `QUEUED`, `STATUS` global signals (56-61); thread-locals `STARTED` (addr -> loop generation), `BUSY`, `HELD`, `ROWS` (63-73, 731-735). `Status` fields (39-53).
- `start` -> `begin` -> `run` (101-195): once per page and identity; retries forever, pause `2000 ms << min(attempt,5)` (max 64 s, line 191). After each await `checkpoint` compares the generation, so a deleted or deleted-and-restored identity's loop stops (126-137).
- `ensure_for` (363-406): `net::note_own`; read the `InboxV2` record; `net::get(addr, subscribe=true)` where an error is *unknown* and aborts (377); record present -> `republish_for`; else `CreateInboxV2 {addr, ghost_master, existing}` (`existing` = state already on the network, i.e. a key-only restore); `Conflict` -> republish path (396-400); then `publish`.
- `publish` (414-423): `net::put`, then `ops::run_jobs()` so the delegate subscribes the inbox; a failed RunJobs is only logged.
- `republish_for` (433-450): nothing if on the network; else `RepublishInboxV2` then `publish`.
- `refresh` (471-518): skipped while `HELD`; coalesced through `BUSY` (a change during a run triggers one more round); `drain` then `load_inbound_for`.
- `drain` (528-564): up to 16 rounds of GET -> `note_owner` -> (only if some message is not tombstoned, `undrained` 521-526) `UpdateInboxV2 {Drain}` -> `net::update(delta)`. `Got::Missing` republishes once per drain because each republish signs a new owner version (536-543). Ends when `remaining == 0 && update.is_none()`.
- `note_owner` (610-634): sets `quarantine`, `waiting_strangers`, sticky `queue_was_full` (`>= QUARANTINE_SLOTS`, 16), `strangers_dropped`, sticky `foreign_version`. `foreign()` = `record.foreign_seen.is_some() || state_version > record.max_version` (604-606); the record is read after the state.
- `read_inbound`/`parse_inbound` (639-676): list `ItemKind::Inbound`, decode `InboundMessage` then `MailBody`; failures counted into `unreadable_mail`.
- `load_inbound_for` (686-720): computes the queue set; if that read fails, every quarantine message is held back (703-714).
- Rows: `row_id` = `local_state::message_id_of(id[..16])`, so read marks and archive work as for v1 (740-750); `ROWS` never forgets within a page. `archive_row` deletes the drained copy only after the archived copy is stored and returns `(stored?, why)` (760-769); `delete_row` -> `local_state::delete_message_after_copies` (kept, archived, drained copy, then the deleted flag) (777-784).
- `forget` (331-345), called from `identity_store::forget_everywhere` (identity_store.rs:371-379).
- Offline-only fixtures `?v2-foreign-fixture`, `?v2-queue-fixture` (197-323); `another_node_signs_with` re-signs the owner record with keys from the exported recovery key (284-323).

### net.rs (1119)
- `Got::{State, Missing}`, `NetError::{Failed, Unknown}` (27-51). `OWN` routes own-inbox changes (73-94).
- `node` (use-node, 101-527). Answers carry no request id, so one request per `(instance id, Kind::{Get,Put,Update})` is out at a time (`Slot`, `Turn`, 131-344). `TIMEOUT_MS = 30_000` (113) -> `Unknown`; a timed-out or cancelled-after-send request keeps its turn for `LATE_MS = 5 min` (126, 321-343) and its late answer is swallowed (`answer`, 199-222). Waiting for a turn has no timeout of its own (460).
- `route` (348-390), called first in `api.rs::handle_response` (ui/src/api.rs:2579-2583): `GetResponse`/`PutResponse`/`UpdateResponse` resolve the slot or are swallowed if the id is `KNOWN`; `UpdateNotification` on an own inbox -> `owner::changed`; `SubscribeResponse` swallowed if known; `NotFound` only when a v2 GET waits; `ContractError::{Get,Put,Update}` -> `Failed(cause)`.
- Requests: `get` = `ContractRequest::Get {return_contract_code: !known(id), subscribe, blocking_subscribe: false}` (475-489); `KNOWN` is set only after the node took a request (413-430, #191). `get_id` (Ghost Key pointer record, no code, no follow) (493-501). `put` = full `ContractContainer` + state, `subscribe: true` (504-516). `update` = `UpdateData::Delta` on `key(addr)` (520-526).
- **There is no Subscribe request and no unsubscribe**: following is the `subscribe` flag on GET (own inboxes and saved v2 contacts, `contacts::follows`, contacts.rs:98-100) and on PUT. Nothing polls: the page learns of new mail from `UpdateNotification` or its own next GET.
- `offline` (!use-node, 828-1118): inboxes in `STATES`, mirrored to the in-page delegate with `local::set_contract_state` (933-944). `merge` (1012-1030): owner record replaced if version higher, time if `owner_time` higher, `revoked`/`usage` unioned, messages first-wins by id, then `filter` (1081-1118): drops tombstoned ids, pass messages whose `pass_id` is revoked, quarantine messages that break the mode (Open refuses a proof; RequireGhost needs one that `proven` accepts, 1049-1079: requestor = `published-contract/contract-id.txt`, payload = `ghost_admit_bytes`, signature, chain to `ghost_master()`, action `freenet-donation`, amount >= tier), then keeps 16 quarantine messages by `blake3(blake3(time) ++ id)` rank. **Not checked** (816-827): pass signatures, certificate expiry, epochs, send window, `per_invite`, per-Ghost-Key cap, 88-message cap, owner signatures. Test hooks: `slow_updates`, `lose_next_put`, `subscribed`, `forget_own`, `store_unannounced`.

### contacts.rs (632)
- `resolve`/`resolved_from` (65-92): GET, decode, require `address_of(owner_vk) == addr`; returns keys, issuer key, quarantine mode, `waiting`, `drained` counts. No signature check here.
- `is_v2(contact)` = the contact has `contact_store::V2Notes {label, invited}` (103-105; ui/src/app/contact_store.rs:31-56, stored in the contact record's `v2` field, :89). v2 contacts live in the v1 address book.
- `add` (178-279): parse; refuse legacy and own address; resolve; for an invite `invite_is_theirs` (cert signed by the inbox's `issuer_vk`, `pass_sk` matches `pass_vk`, 303-322) then `Op::HoldPass`; own identity -> nothing saved; an existing contact keeps its name and gains notes; a new `Contact` gets `inbox_wasm_hash = Some(v2 CODE_HASH)`, `Tier::Day1` (247-258); notes are set only after `insert_contact` succeeded; `contact_store::save` before success.
- Pass display: page memory `HELD`/`LAPSED` (38-44) fed by `note_drained`, `note_send` (326-355), plus persisted `notes.invited`; `held()` (366-379). The delegate never shows held passes. Issued passes come from the ledger: `ledger`/`parse_ledger` (one undecodable entry fails the read, 402-429), `issued_to` -> `NotYet | Live {expiry, ids} | Revoked` for `PassClass::Contact` only (431-447).
- `revoke` (451-461) and `change_own` (465-488): GET, `UpdateInboxV2 {change}`, UPDATE, up to 16 rounds while `ChangeOutcome::WaitsForDrain`; `WaitsForStorage` is an error.
- `create_invite` (492-515): `IssuePass {contact: None, invite: Some(n)}` -> `address::encode_invite`.
- `rows`, `sender_name`, `petnames` (127-134, 528-555).

### queue.rs (227): new-contacts queue view
- A drained message is *queued* if `admission == KIND_QUARANTINE`, its id is not in the identity's accepted list, and the ledger shows no live pass for its sender (103-119). Accepted ids: hex strings in field `v2_accepted` of the `IdentitySettings` JSON item (beside `"migration"`), capped at `ACCEPTED_MAX = 1024` oldest-first (20-23, 66-100). Undecodable settings are an error, never "none" (37-51).
- `accept` (141-187): send `SealContent::PassUpdate` to the sender; only if delivered, record ids, then add them as a contact under the typed petname (default: first 8 chars of the address). `drop_message` (191-194) = `delete_row`; the sender is told nothing.

### send.rs (587)
- `send_and_record` (437-507): writes a `SentMessage` (`Pending`) to local state, then in `spawn_forever` runs `send_waiting`, sets `Delivered`/`Failed`, toasts `Outcome::text`, sets `LAST_OUTCOME`.
- `send_waiting` (196-324), per round: GET recipient (`Missing`/error -> `Unreachable`, nothing sent); if RequireGhost and a key was chosen, fetch its vk; `Op::SealV2 {.., ghost_vk, reseals}`; a failure whose `detail` contains "requires a Ghost Key" -> `NeedsGhostKey` (231); if `ghost_payload` came back, have the vault sign it; a different signing key re-seals without counting (`MAX_KEY_CHANGES = 3`, 256-275, 327); tier below need -> `BelowTier`; attach the proof; UPDATE with a one-message `InboxDelta` (errors only logged, 291-295); `read_back` 4 times 1.5 s apart (167-168, 387-400) using `w::delivered` (in `messages` or tombstoned, protocol/src/inbox_v2.rs:332-335).
- Not read back: notice `QueueFull`/`PassFull` end the send (306-309). Otherwise `wait_for` (336-351) re-sends the same delta and re-reads with pauses 5 s doubling to 10 min until `reseal_at`, then seals anew; after `MAX_RESEALS = 4` (protocol/src/lib.rs:438) -> `AfterReseals`. The user gets one warning toast when waiting starts (475-483).
- Texts (71-118): lapsed pass -> "Delivered to X's new-contacts queue: the pass you held for them has lapsed..."; full queue -> "Not delivered: X's new-contacts queue is full. A Ghost Key, or an invite from them, gets you in."; full pass -> "X has not read your earlier messages yet...".
- `check_recipient` (406-433): the v1 safeguards (recorded recipient binding, "Require known key to send"). `GHOST_CHOICE` is per recipient (50-59), cleared when compose mounts (app.rs:5963).

### address.rs (628)
- Address text = base58(`addr24 ++ check2`), `check2 = blake3::derive_key("freenet-mail address check v1", addr)[..2]`, left-padded with `1` to 36 chars (15-39). `parse_address` accepts 35/36 chars or a legacy 32-byte id (>= 43 chars) (82-106).
- `Invite {v:1, label, pass}` = `fmail-invite-1:` + base58(CBOR) (21, 111-138). `parse_recipient`: invite, `fmail:` prefix (case-insensitive), label split at the last `@` (176-199).
- Labels: `normalize_label` = NFKC, lowercase, NFKC; 1-32 chars; no `@`, whitespace, control or format chars; incoming labels ASCII only, petnames one script (262-290). `lookalike_key` + TR39 skeleton (294-311). `name_for` (339-374): petname wins, warnings `DiffersFromPetname`, `LooksLike`; fallback name is the first 8 address chars.

### ghostkeys.rs (1086)
- API: `presence` (HasIdentity), `keys` (ListGhostKeys + GetDefaultKey), `request_access` (RequestAnyAccess), `verifying_key` (GetCertificate fallback), `sign` (SignWithDefault/SignMessage) (101-190). `proof` verifies the certificate against `v2::ghost_master()` and requires action `freenet-donation` (192-229). `answers` binds the reply to this payload, this app (`WEBAPP_ID` = `published-contract/contract-id.txt`, 23, 300-309) and the certified key (235-261).
- `node::ask` (584-634): resolves the vault's delegate key through ghostkeys' pointer record with `freenet_migrate::PointerResolver` (`AUTHOR_VK`, `APP_ID = "ghostkeys.ghostkey-delegate"`, `FALLBACK_CODE_HASH` for `NeverPublished`; floor in localStorage `fmail-ghostkeys-pointer-floor`) (338-349, 417-489); one request at a time; `QUICK_MS = 30 s`, `PROMPT_MS = 10 min`, `LATE_MS = 5 min`. Replies arrive via `api.rs:3421-3423` (`takes`, `response`) and `api.rs:2592-2594` (`missing`).
- `ask` selection (311-319): `mock::ask` whenever `test-ghost-master` is on (even with `use-node`); `node::ask` for plain `use-node`; a stub error otherwise.
- `mock` (753-981): fixture keys of `contracts/inbox-v2/tests/fixtures/ghost-keys.txt` (indices 0-3 $1, 4-7 $5, 8-9 $20), state in localStorage `fmail-ghostkeys-mock`; `MARKER = "freenet-mail:test-ghost-master-built-in"` (762-766), kept in the WASM by `log::info` in ui/src/lib.rs:161-162; `scripts/check-release-scrub.sh:45,56-61` greps for it.
- `get_link` -> `https://freenet.org/ghostkey/create/?return_to=<webapp id>&return_path=%23%2Fcompose%3Fdraft%3D<id>` (271-286); `draft_in_route` (290-297) consumed at app.rs:1466-1478.

### app/v2_view.rs (911)
`ForeignNodeWarning` (27-48, also the unreadable-mail notice); `ScrAddress` (51-301: address + copy, inbox state, quarantine mode buttons Anyone / Ghost Key / $5 / $20, queue, invite of 3 messages, add contact, contact rows); `QueueView` + `PetnameForm` (307-511); `QueueNotice` (515-527); `ContactRow` with Revoke (529-605); `GhostPanel` (610-806: presence, picker, tier warning, disclosure text, "Get a Ghost Key" after saving the draft); `RecipientNote` (810-893, advisory, exposes `data-waiting`, `data-drained`, `data-held`); `LastOutcome` (896-911).

## 3. Interfaces

- **Delegate ops:** `CreateInboxV2` (owner.rs:385), `RepublishInboxV2` (441), `UpdateInboxV2` (572), `RunJobs` (416), `HoldPass` (contacts.rs:207), `IssuePass` (497), `SealV2` (send.rs:370). **Items read/written:** `InboxV2` (owner.rs:366, 588), `Inbound` (640, 787; deleted with generation), `PassLedger` (contacts.rs:403), `IdentitySettings` (queue.rs:25-31), contacts via `contact_store`.
- **Node API:** `ContractRequest::{Get, Put, Update}`, `DelegateRequest::ApplicationMessages` to the ghostkeys delegate.
- **Called from:** app.rs (1368-1462 rows; 3393 start; 3744-3746 `data-v2-stored`; 4364 reply address; 5277-5300 archive; 5378-5391, 5402 delete; 5575 add sender; 6175-6240 send; 6767-6782 note), identity_store.rs:376, 850; settings.rs:192; api.rs:2582, 2594, 3421; lib.rs:162.
- **Features** (ui/Cargo.toml:100-122): `use-node` (default), `no-sync` -> `local-delegate`, `example-data` -> `local-delegate` + `test-ghost-master`. Pinned deps `ghostkey-verify =0.1.0`, `ghostkey-common =0.4.0` (52-55).
- **Formats this code owns:** address text, invite text, `MailBody`, `v2_accepted`, `V2Notes`. Everything else is `protocol/src/inbox_v2.rs` (sibling map protocol-wire).

## 4. Invariants and where they are held

1. Unknown is never absent: nothing is created or sealed on an unanswered GET (owner.rs:151-152, 376-380; net.rs:7-12). Tests: `net::node::tests` (net.rs:545-812).
2. An answer resolves only the request it belongs to (net.rs:131-344; ghostkeys.rs:491-634). Tests `a_late_answer_is_the_timed_out_requests...` (net.rs:678), `a_late_vault_answer_is_never_the_next_requests` (ghostkeys.rs:690).
3. First GET of an inbox asks for its code, and only a GET the node took counts (net.rs:413-430, 480-489; tests 546, 794).
4. A record with no inbox on the network is republished; at most once per drain (owner.rs:433-450, 536-543; flows.rs:333, 361).
5. `RunJobs` after every PUT that lands, none after one that failed (owner.rs:414-423; flows.rs:303).
6. A deleted identity's loops stop and leave nothing (owner.rs:126-137, 331-345; tests 819, 934, 960, 990).
7. Delivery is decided by the read-back, never by the UPDATE's answer (send.rs:291-303); no re-seal before `reseal_at` (311-322; flows.rs:589, 601).
8. A Ghost Key proof is attached only when the delegate sealed for a RequireGhost queue; a key chosen for one recipient is never used for another (send.rs:55-59, 251; flows.rs:755; send.rs test 544).
9. Undecodable stored data is reported, not treated as empty: mail (owner.rs:655-676, test 869), ledger (contacts.rs:412-429, test 593), settings (queue.rs:37-51, tests 203, 216).
10. Delete order: copies first, deleted mark last (owner.rs:771-784; flows.rs:139, 164, 201, 225).
11. A stranger's label never becomes a petname unseen; lookalikes are flagged (queue.rs:124-136, 156-163; address.rs:339-374; flows.rs:852; app.rs:8064, 8109).
12. An invite is never a recipient and never stored in Sent (contacts.rs:136-145; app.rs:6175-6182).
13. Only own inboxes and saved contacts are followed (contacts.rs:94-100; flows.rs:832).
14. The page's inbox key equals the delegate's (mod.rs:121-131).
15. Offline tests prove nothing about contract rules (net.rs:816-827; flows.rs:6-9).

Mutation guards in `scripts/check-guards-fail.sh:1244-1314` (13 named "inbox v2 UI: ..." / "ghost keys: ...") back items 2-6, 9, 10. They patch exact source strings.

## 5. State

- Candidate, not frozen: the contract bytes the UI embeds (mod.rs:26-29). Release blocked (section 1).
- Nothing in these files is frozen bytes. De facto wire formats once users exchange them: address text (the contract has its own `signed::address_check`), invite text, `MailBody`.
- Unfinished: legacy-address resolution ("C2c", address.rs:76-78; contacts.rs:181-187); no resume of an interrupted send; `owner::state_seen` is an empty hook (owner.rs:465-467).

## 6. Tests

- `cargo make test-ui-offline` = `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` (Makefile.toml:916-919). Runs `flows.rs` (31 tests against the real delegate code in-page plus the offline inbox model: create/republish, archive/delete copies, invites, first contact and passes both ways, revocation -> lapsed, RequireGhost with and without proof, 17 strangers into 16 slots, decision 25, slow and lost updates on a fake clock (`ops::pause_ms` advances `now_ms` natively, ui/src/mail_delegate/ops.rs:606-626), queue accept/drop, follow rule, petnames, recipient safeguards, add sender); the gated tests in owner.rs (934-1010), queue.rs (216); mock-vault tests (ghostkeys.rs:999, 1042). Only `v2::flows`: add `v2::flows`.
- `cargo test -p freenet-email-ui --lib v2::net` and `... --lib ghostkeys::node` (default `use-node`; check-guards-fail.sh:134-146): the slot/late-answer tests with hand-fired timers. Also in `cargo test --workspace`.
- Any feature set: address.rs (14 tests), contacts.rs (3), send.rs (3), mod.rs (3), owner.rs pure tests (819, 869, 1013, 1026), `app::v2_sender_tests` (app.rs:8019+).
- Playwright offline (`cargo make test-ui-playwright`, needs `dx`, browsers): `ui/tests/inbox-v2.spec.ts` (9 tests) and `inbox-v2-ghost.spec.ts` (5 tests); both identities live in one page, so contact rows and revocation between two people are not covered there (inbox-v2.spec.ts:6-9).
- `cargo make test-e2e-inbox-v2` (Makefile.toml:1133-1196): builds with `FREENET_UI_FEATURES=test-ghost-master`, two iso nodes, asserts `check-release-scrub.sh` refuses that build, runs `inbox-v2-live.spec.ts` (one test; `NOTIFY_BOUND_MS = 120_000`, `TAB_CLOSED_WAIT_MS = 1_100_000`). No workflow under `.github/workflows` runs it.

## 7. Changing it safely

- UI-only edits here need no candidate rebuild. Edits to `protocol/` or the contract do (AGENTS.md), and change `CODE_HASH`, so every inbox id moves.
- Before a release: freeze the contract and delegate under `published-contract/wasm/v2/` and repoint mod.rs:27-29, or `check-release-delegate.sh` fails.
- Guards match literal text in owner.rs, net.rs, contacts.rs, queue.rs, ghostkeys.rs and app.rs; reformatting a guarded line makes the guard fail to apply. Run `scripts/check-guards-fail.sh --list` and the affected guard.
- `MailBody`, `InboxRecordView`, `LedgerView`: add fields with `#[serde(default)]`. A `MailBody` that no longer decodes old bodies turns stored mail into "could not be read". Only `foreign_seen` has a name-pinning test against the delegate's struct (flows.rs:624-643).
- `v2_accepted` shares the `IdentitySettings` item with the migration markers: write only through `ops::put_item_merged` (queue.rs:68).
- The `"requires a Ghost Key"` substring (send.rs:231) must stay in the delegate's refusal text (delegates/mail-delegate/src/inbox_v2.rs:2004).
- `WEBAPP_ID` must equal the contract's `WEBAPP_REQUESTOR` (contracts/inbox-v2/src/merge.rs:76-78).
- Marker string: keep ghostkeys.rs:762-766, lib.rs:161-162 and check-release-scrub.sh:45 in step.
- Never present an offline (`flows.rs`, Playwright offline) result as proof of signatures, expiry, epochs, window or caps.
- New `data-testid`s go in three places: `ui/src/testid.rs:201+` (`FM_V2_*`), `ui/branding/testids.json` and the `TestIds` interface in `ui/tests/testids.ts`; `testid::tests::json_matches_rust_consts` (ui/src/testid.rs:389+) fails on a mismatch.

## 8. Docs/comments vs code

1. AGENTS.md "`example-data` turns it on, so every offline build has it": a `no-sync`-only build is offline without `test-ghost-master` (ui/Cargo.toml:104-108; mod.rs:7-8); it then names the production master and its Ghost Key calls return "this build has no node" (ghostkeys.rs:316-319).
2. AGENTS.md says the scrub check "refuses to publish such a build to production"; the script refuses any publish, test ones included, unless `--absent` with `FREENET_TEST_GHOST_ISO=1` (check-release-scrub.sh:40-61).
3. AGENTS.md "waits up to 18.3 more minutes": the spec waits until 1,100,000 ms after the send in total (inbox-v2-live.spec.ts:47, 387).
4. ghostkeys.rs:491-492, 607-608 say a cancelled call keeps the vault's turn like a timed-out one. `Owed.until` is `now_ms()` until the select finishes (609-612, 624), so a call dropped mid-wait leaves an already-expired entry and the next request goes out at once (561-581).
5. send.rs:478-479 toast "This page keeps checking while it is open; Sent shows it as pending until then": on a node build a reload marks any Sent row pending for over 90 s as failed (ui/src/inbox.rs:254, 323-345; identity_store.rs:878).
6. AGENTS.md "`net.rs` is the GET, PUT and UPDATE": accurate; note there is no explicit subscribe call (section 2).
7. `published-contract/candidate/README.md` says nothing embeds `inbox_v2` (sibling map inbox-v2-contract); mod.rs:27 does.

## 9. Suspected weak spots

- (medium) Item 8.4: a cancelled Ghost Key request's late answer can be handed to the next request. `sign` is protected by `answers`; `presence`/`keys` would show an "unexpected answer" error. `GhostPanel`'s `use_resource` restarts (`checks`, v2_view.rs:621-629) are the realistic trigger. Only the timeout path is tested.
- (medium) v2 sends are not in `inbox::in_flight_sent()` (ui/src/inbox.rs:299-311, v1's `PENDING_SENT_ACK` only). `load_identities` runs again after a migration or confirm (identity_store.rs:811-813) and calls `fail_stranded_pending_sent`; a v2 send still waiting after 90 s is marked Failed locally, and terminal states stick (ui/src/local_state.rs:811-832), while send.rs:490 discards that result and still writes the final state to the delegate.
- (medium) A send interrupted by closing the tab is never resumed; decision 9's queued send lives only in the page task (send.rs:466).
- (medium) New v2 contacts get `inbox_wasm_hash = v2 CODE_HASH` (contacts.rs:256). The v1 `prime_contact_inbox` (ui/src/api.rs:420-446) derives a v1-params key under that hash and sends a subscribing GET for a contract that cannot exist.
- (low) `contacts::add` with an invite calls `HoldPass` before `insert_contact`; if the insert is refused (name taken) the delegate holds the pass but the page shows none (contacts.rs:203-217, 259-261).
- (low) `queue::accept` awaits the full `send_content` (queue.rs:143), so a PassUpdate that never reads back keeps every Accept/Drop button disabled until `reseal_at` rounds end.
- (low) No periodic refresh: if the node drops the subscription, new mail shows only after a reload or another own GET.
- (low) Coupling by error substring (send.rs:231).

## 10. Open work

- Legacy 44-char address to v2 address ("C2c", address.rs:76-78); old contacts "move over in a later update" (contacts.rs:183-186).
- Freeze the contract at v0.3.0 (mod.rs:26).
- `state_seen` placeholder (owner.rs:465-467).
- Live spec does not cover pass renewal (inbox-v2-live.spec.ts:33-36).
- `queue_was_full` and held/lapsed pass state are page memory, lost on reload (owner.rs:47-49; contacts.rs:38-44).

## 11. Questions

1. Is creating a v2 inbox for every confirmed identity on first load of v0.3.0 the intended rollout, with no opt-in?
2. After `net::update` of a drain, can a real node's next GET still show the undrained state, and does `UpdateInboxV2` then sign further owner versions inside the 16-round loop (owner.rs:531-562)?
3. Does a node resend `UpdateNotification` after reconnect, or must the page re-GET with `subscribe` to keep following?
4. Should a v2 contact carry a v1 `inbox_wasm_hash` at all?
5. A test-key publish with the real ghostkeys vault gets proofs for another requestor; `answers` then reports "Ghost Key answered another request" (ghostkeys.rs:249-258). Intended?

---

## Appendix: the reader's structured summary

The page-side driver of inbox v2 (about 7,400 lines). It signs and seals nothing: it calls the mail delegate (CreateInboxV2, RepublishInboxV2, UpdateInboxV2, SealV2, HoldPass, IssuePass, RunJobs), moves contract states between delegate and node, and reports outcomes.

v2 has no feature gate. Every confirmed identity gets a v2 inbox at page load, published from the candidate contract bytes, next to its v1 inbox. Received v1 and v2 mail share one list; unaccepted stranger mail waits in a Settings queue. Compose routes by the To text alone: a 36-character address or a v2 contact's name goes v2, anything else goes v1 with AFT. There is no fallback either way, and legacy 44-character addresses cannot be added as v2 contacts.

No production user sees any of this yet: releases are frozen, and the release check fails while the UI embeds the candidate.

On a node, net.rs allows one request per inbox and kind at a time, treats a timeout as unknown, and holds the turn 5 minutes for a late answer. Following an inbox is only the subscribe flag on GET and PUT; nothing polls. The offline model checks a small subset of contract rules and says so.

Sending reads back after every update, waits until reseal_at before sealing again (4 reseals at most), and tells the user plainly about a lapsed pass, full queue or full pass. Ghost Keys go through the real vault via a pointer record, or a mock vault under test-ghost-master.

I ran no tests (read-only). The map file is complete on disk, so a restart can reuse it.

### Key facts

- v2 is not feature-gated: owner::start runs for every confirmed identity at load and on each UserInbox render (ui/src/app/identity_store.rs:847-851, ui/src/app.rs:3389-3395); unconfirmed identities publish nothing.
- The UI embeds the candidate contract, not a frozen one: CODE/CODE_HASH include published-contract/candidate/inbox_v2 (ui/src/v2/mod.rs:26-29). scripts/check-release-delegate.sh:72-75 fails a release while ui/ references the candidate, and .release-freeze:1-11 blocks releases anyway.
- Inbox identity = code hash + params_bytes(addr, ghost_master); ghost_master is the fixture master under test-ghost-master, else the Freenet production master (ui/src/v2/mod.rs:35-53). The page's key must equal the delegate's (test at mod.rs:121-131).
- Inbox creation: ensure_for reads the InboxV2 record, GETs with subscribe, then CreateInboxV2 (existing state passed for a key-only restore) or the republish path on an existing record or Conflict; publish = PUT then RunJobs (ui/src/v2/owner.rs:363-450). A failed GET is 'unknown' and aborts; the start loop retries forever with backoff up to 64 s (owner.rs:149-195).
- Drain: up to 16 rounds of GET, note_owner, UpdateInboxV2{Drain} only if some message is untombstoned, then UPDATE with the delta; a Missing inbox is republished once per drain (ui/src/v2/owner.rs:521-564). Drained mail is then listed from the delegate's Inbound items (owner.rs:639-720).
- Receiving coexistence: v1 rows are built first, then merge_v2_rows swaps in owner::inbound(addr,false) (ui/src/app.rs:3777-3784, 1527-1530). v2 row ids are local_state::message_id_of(first 16 bytes of the message id) so read marks and archive reuse v1 local state (ui/src/v2/owner.rs:740-750).
- Sending coexistence: compose sends v2 only when contacts::recipient_addr resolves (address forms or the exact name of a contact with V2Notes); invites are refused; everything else takes the v1 AFT path (ui/src/app.rs:6175-6240, ui/src/v2/contacts.rs:148-163). contacts::add refuses legacy 44-char addresses (contacts.rs:181-187).
- Node transport: one request out per (instance id, Get|Put|Update); TIMEOUT_MS 30 s gives NetError::Unknown; a timed-out or cancelled request keeps its turn LATE_MS = 5 min and its late answer is swallowed (ui/src/v2/net.rs:113, 126, 131-344). route() is called first in api.rs handle_response (ui/src/api.rs:2579-2583).
- There is no Subscribe request and no unsubscribe: following is the subscribe flag on GET (own inboxes and saved v2 contacts only, ui/src/v2/contacts.rs:94-100) and subscribe:true on PUT (ui/src/v2/net.rs:475-516). The first GET of an inbox asks for contract code, and only once the node took a request is the inbox 'known' (net.rs:413-430).
- Offline model (ui/src/v2/net.rs:816-1118): checks newest owner version, unions, tombstones, revocation by pass id, the Ghost Key proof and 16 quarantine slots; does not check pass or owner signatures, expiry, epochs, send window, per_invite, per-Ghost-Key cap or the 88-message cap. Offline tests are not proof of those.
- Send loop (ui/src/v2/send.rs:196-324): GET, SealV2, optional Ghost Key signature, UPDATE (its answer never decides), 4 read-backs 1.5 s apart, then wait_for re-sends the same delta with 5 s to 10 min pauses until reseal_at, re-seal, up to MAX_RESEALS = 4 (protocol/src/lib.rs:438). QueueFull and PassFull notices end the send at once (send.rs:306-309).
- User-facing outcomes are in Outcome::text (ui/src/v2/send.rs:71-118): lapsed pass is 'Delivered to X's new-contacts queue: the pass you held for them has lapsed', full queue is 'Not delivered: ... queue is full. A Ghost Key, or an invite from them, gets you in'.
- New-contacts queue: a drained message is queued if admission is quarantine, its id is not in the v2_accepted list of the IdentitySettings JSON (max 1024) and its sender has no live ledger pass (ui/src/v2/queue.rs:20-23, 103-119). Accept sends a PassUpdate first and records only if delivered (queue.rs:141-187).
- Pass display: held/lapsed state is page memory plus persisted V2Notes.invited (ui/src/v2/contacts.rs:38-44, 326-379; ui/src/app/contact_store.rs:31-56); issued passes come from the PassLedger items, and one undecodable entry fails the read (contacts.rs:402-447).
- Addresses: base58(addr24 + 2-byte blake3 check, context 'freenet-mail address check v1') padded to 36 chars; invites are 'fmail-invite-1:' + base58 CBOR of {v,label,pass}; labels are NFKC-casefolded, 1-32 chars, incoming ASCII only; a petname always wins (ui/src/address.rs:15-39, 111-138, 262-290, 339-374).
- Ghost Keys on a node: the vault's delegate key is resolved through ghostkeys' pointer record with freenet_migrate::PointerResolver, one request at a time, 30 s or 10 min timeouts (ui/src/ghostkeys.rs:338-349, 448-489, 584-634). Proofs are bound to WEBAPP_ID = published-contract/contract-id.txt, which the contract pins (ghostkeys.rs:23, 235-261; contracts/inbox-v2/src/merge.rs:76-78).
- test-ghost-master: mock::ask replaces the node vault even with use-node (ui/src/ghostkeys.rs:311-314); marker 'freenet-mail:test-ghost-master-built-in' (ghostkeys.rs:762-766) is kept in the WASM by ui/src/lib.rs:161-162 and grepped by scripts/check-release-scrub.sh:45,56-61. example-data enables it; no-sync alone does not (ui/Cargo.toml:104-108).
- Tests: cargo make test-ui-offline runs flows.rs (31 tests, real delegate code in-page, fake clock via ops::pause_ms, ui/src/mail_delegate/ops.rs:606-626); net and ghostkeys node tests need default use-node features (scripts/check-guards-fail.sh:134-146); cargo make test-e2e-inbox-v2 is two real nodes (Makefile.toml:1133-1196) and is in no CI workflow.
- 13 mutation guards patch literal source text in owner.rs, net.rs, contacts.rs, queue.rs, ghostkeys.rs and app.rs (scripts/check-guards-fail.sh:1244-1314); rewording a guarded line breaks its guard.
- MailBody (ui/src/v2/mod.rs:57-81) is a UI-private CBOR format with no golden bytes; InboxRecordView and LedgerView are partial reads of delegate items, and only foreign_seen has a name-pinning test (ui/src/v2/flows.rs:624-643).

### Docs or comments that disagree with the code

- **[low]** Claim (AGENTS.md, 'Inbox v2 in the UI', test-ghost-master bullet): `example-data` turns test-ghost-master on, so every offline build has it **Reality:** A no-sync-only build is offline without test-ghost-master: it names the production master and its Ghost Key calls return 'this build has no node'. The module comment in mod.rs says so itself. Evidence: ui/Cargo.toml:104-108; ui/src/v2/mod.rs:7-8; ui/src/ghostkeys.rs:316-319
- **[low]** Claim (AGENTS.md, 'Inbox v2 in the UI', test-ghost-master bullet): check-release-scrub.sh refuses to publish a test-ghost-master build to production **Reality:** The script refuses any publish carrying the marker, test publishes included, unless run with --absent and FREENET_TEST_GHOST_ISO=1. Evidence: scripts/check-release-scrub.sh:40-61
- **[low]** Claim (AGENTS.md, test-e2e-inbox-v2 bullet (and Makefile.toml:1134 description)): When the notification does not drain, the run waits up to 18.3 more minutes for the rescan wake-up **Reality:** The spec waits until 1,100,000 ms after the send in total, so about 16.3 minutes beyond the 120 s notification bound. Evidence: ui/tests/inbox-v2-live.spec.ts:45-47, 378-390
- **[medium]** Claim (code comments, ui/src/ghostkeys.rs:491-492 and 607-608): A Ghost Key request that timed out or whose caller stopped waiting keeps the vault's turn for LATE_MS, and its late answer is left for the next request to wait on **Reality:** Owed.until is set to now_ms() at send and only moved to now+LATE_MS after the select returns, so a call dropped mid-wait leaves an already-expired entry; wait_for_late gives up immediately and the next request goes out at once. Only the timeout path holds the turn. Evidence: ui/src/ghostkeys.rs:609-624, 561-581
- **[medium]** Claim (user-facing toast, ui/src/v2/send.rs:476-480): Sent to X, but not in their inbox yet. This page keeps checking while it is open; Sent shows it as pending until then **Reality:** On a node build, any identity reload runs fail_stranded_pending_sent, which marks Sent rows pending for over 90 s and not in the v1 in-flight set as failed; v2 sends are never in that set. Evidence: ui/src/inbox.rs:254, 299-345; ui/src/app/identity_store.rs:878
- **[medium]** Claim (AGENTS.md, 'Frozen artifacts' and 'Inbox v2'): The UI embeds published-contract/wasm/v1/*; inbox v2 is 'the next inbox' used 'alongside v1' **Reality:** Accurate but incomplete: the UI also embeds the inbox v2 candidate, and creates a v2 inbox for every confirmed identity with no opt-in or feature gate. Evidence: ui/src/v2/mod.rs:27-29; ui/src/app/identity_store.rs:847-851
- **[low]** Claim (code comment, ui/src/address.rs:76-78): A legacy 44-character id is recognised and 'C2c resolves it to the address of the key behind it' **Reality:** Nothing resolves it: contacts::add refuses a legacy address and recipient_addr returns None for one, so it falls to the v1 path. Evidence: ui/src/v2/contacts.rs:148-158, 181-187

### Suspected issues (leads from reading, not confirmed)

- **A cancelled Ghost Key request does not hold the vault's turn** (confidence: medium). Owed.until is now_ms() until the select completes, so dropping the future mid-wait (GhostPanel's use_resource restarts when 'checks' changes) leaves an expired entry. The next request is sent immediately and can receive the earlier request's answer. sign() is protected by answers(); presence() and keys() would surface an 'unexpected answer' error, and answers can stay shifted while requests overlap. Only the timeout path is tested. Evidence: ui/src/ghostkeys.rs:609-624
- **v2 sends are invisible to the stranded-pending sweep** (confidence: medium). in_flight_sent() lists only v1's PENDING_SENT_ACK. load_identities reruns after a migration or confirm and calls fail_stranded_pending_sent, so a v2 send still waiting after 90 s is marked Failed locally; terminal states stick, and send.rs discards the result of the later local update while still writing the final state to the delegate, so page and store can disagree until reload. Evidence: ui/src/v2/send.rs:490
- **An interrupted v2 send is never resumed** (confidence: medium). The wait-until-reseal_at loop lives only in a spawn_forever page task. Closing the tab ends it; the next load marks the Sent row failed (node builds) and nothing re-sends or re-checks it. Evidence: ui/src/v2/send.rs:466
- **New v2 contacts carry the v2 code hash in the v1 inbox_wasm_hash field** (confidence: medium). contacts::add stores inbox_wasm_hash = v2 CODE_HASH. The v1 prime_contact_inbox derives a v1-params contract key under that hash and sends a subscribing GET for a contract that cannot exist, for each such contact. I did not trace every caller of the prime path. Evidence: ui/src/v2/contacts.rs:256
- **An invite's pass can be held without a contact** (confidence: low). add() calls HoldPass before address_book::insert_contact. If the insert is refused (name taken by another key), the delegate keeps the pass but no notes or HELD entry are written, so the page shows no pass although mail would go by pass. Evidence: ui/src/v2/contacts.rs:203-217
- **Accept can block the whole queue view for a long time** (confidence: low). queue::accept awaits the full send of a PassUpdate, including the wait-until-reseal_at rounds, while the view's shared 'busy' flag disables every Accept and Drop button. It needs a reachable inbox that silently drops the message to trigger. Evidence: ui/src/v2/queue.rs:143
- **No periodic refresh of the own inbox** (confidence: low). The page drains only on an UpdateNotification, its own PUT/UPDATE flows, or user actions. If the node drops the subscription, mail drained by the delegate in the background shows only after a reload. Evidence: ui/src/v2/owner.rs:453-463
- **NeedsGhostKey is detected by an error substring** (confidence: high). send_waiting matches the delegate's refusal on detail.contains("requires a Ghost Key"). Rewording the delegate's message turns the outcome into a generic 'could not be sealed'. Evidence: ui/src/v2/send.rs:231

### Open work

- Legacy 44-char address to v2 address resolution ('C2c') is not implemented; old contacts 'move over in a later update' (ui/src/address.rs:76-78, ui/src/v2/contacts.rs:181-187).
- The contract is still the candidate; 'v0.3.0 freezes it' and the UI must then point at published-contract/wasm/v2/ (ui/src/v2/mod.rs:26-29, scripts/check-release-delegate.sh:72-75).
- owner::state_seen is an empty hook (ui/src/v2/owner.rs:465-467).
- queue_was_full and held/lapsed pass state are page memory only, lost on reload (ui/src/v2/owner.rs:47-49, ui/src/v2/contacts.rs:38-44).
- The two-node spec does not cover a pass renewal reaching a contact (ui/tests/inbox-v2-live.spec.ts:33-36).
- cargo make test-e2e-inbox-v2 is not run by any workflow under .github/workflows (Makefile.toml:1133-1196).
- Offline model deliberately omits pass signatures, expiry, epochs, send window and most caps (ui/src/v2/net.rs:816-827).

### Open questions

- Is creating a v2 inbox for every confirmed identity on first load, with no opt-in, the intended v0.3.0 rollout?
- After the page sends a drain UPDATE, can a real node's next GET still return the undrained state, and does UpdateInboxV2 then sign further owner versions inside the 16-round loop (ui/src/v2/owner.rs:531-562)? I did not read the delegate's handling of an already-drained state.
- Does a node keep or resend subscriptions after a reconnect, or must the page GET again with subscribe to keep following its inbox?
- Should a v2 contact carry any value in the v1 inbox_wasm_hash field, given the v1 prime GET uses it?
- A test-key publish talking to the real ghostkeys vault gets proofs for a different requestor; the page then reports 'Ghost Key answered another request' (ui/src/ghostkeys.rs:249-258). Is that the intended message for this case?
- Nothing here was executed (read-only task): test counts and commands come from reading the sources and scripts, not from a run.

### Coverage

Read in full: `ui/src/v2/mod.rs`, `ui/src/v2/owner.rs`, `ui/src/v2/net.rs`, `ui/src/v2/contacts.rs`, `ui/src/v2/queue.rs`, `ui/src/v2/send.rs`, `ui/src/v2/flows.rs`, `ui/src/address.rs`, `ui/src/ghostkeys.rs`, `ui/src/app/v2_view.rs`, `.release-freeze`

Skimmed: `ui/src/app.rs (v2 call sites only: 1040-1060, 1355-1478, 1527-1549, 3375-3404, 3714-3726, 3735-3893, 4352-4376, 4964-4982, 5268-5300, 5356-5412, 5560-5590, 5955-5966, 6160-6249, 6755-6782, 8015-8110)`, `ui/src/api.rs (412-446, 2262-2290, 2570-2600, 3412-3426, 3834-3854)`, `ui/src/app/identity_store.rs (360-380, 780-912)`, `ui/src/app/contact_store.rs (31-110)`, `ui/src/app/address_book.rs (function headers via grep)`, `ui/src/app/settings.rs (grep)`, `ui/src/inbox.rs (296-345)`, `ui/src/local_state.rs (231-245, 800-875)`, `ui/src/local_state/items.rs (85-99, 588-602)`, `ui/src/mail_delegate/ops.rs (599-626)`, `ui/src/mail_delegate/mod.rs (155-197)`, `ui/src/lib.rs (120-175)`, `ui/src/testid.rs (grep)`, `ui/Cargo.toml (features and ghostkey deps)`, `protocol/src/lib.rs (330-548)`, `protocol/src/inbox_v2.rs (constants, delivered)`, `delegates/mail-delegate/src/inbox_v2.rs (grep for constants and the refusal string)`, `contracts/inbox-v2/src/merge.rs (grep for the requestor pin)`, `scripts/check-release-scrub.sh (30-68)`, `scripts/check-release-delegate.sh (60-80)`, `scripts/check-guards-fail.sh (128-163, 1240-1314)`, `Makefile.toml (505-527, 907-937, 1133-1204)`, `ui/tests/inbox-v2.spec.ts (header and test names)`, `ui/tests/inbox-v2-ghost.spec.ts (header and test names)`, `ui/tests/inbox-v2-live.spec.ts (1-60, 360-423)`, `ui/tests/playwright.config.ts (grep)`, `ui/tests/testids.ts (1-25)`, `docs/qa/manual-test-inventory.md (grep)`, `sibling maps in the scratchpad (grep only)`

Skipped: (none)
