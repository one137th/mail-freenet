> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# ui-app-shell: UI application shell (ui/src/app.rs, testid.rs, branding/, vendor css)

Read at upstream commit 2358b59. Paths are relative to the repo root; `app.rs:N` means `ui/src/app.rs:N`. Nothing was built or run: sections 8 and 9 are from reading.

## 1. Purpose and where this sits

`ui/src/app.rs` (8,125 lines, about 5,000 of them non-test) is the Dioxus 0.7 root component `app()` and the whole signed-in mail experience: the page contexts, the `NodeAction` vocabulary the rest of the UI uses to reach the node loop, the in-memory inbox view model (`InboxView`, `Message`), thread grouping (`mod threads`), a storybook host, and every mail component (top bar, sidebar, list, detail, thread views, compose). Login and identity list, settings, address book, backup, migration and the v2 panels are submodules declared here (app.rs:26-39) under `ui/src/app/`; sibling maps ui-state and ui-v2 cover them.

It owns no wire format and no storage. Durable state goes through `crate::local_state` (mail delegate items), `crate::inbox` / `crate::aft` via `crate::api` (v1 contracts), and `crate::v2::*` (inbox v2). app.rs is where the v1 and v2 paths are chosen, for sending and for display.

## 2. Walkthrough

### 2.1 Root, contexts, screen switching (app.rs:1-406)

- `NodeAction` (app.rs:56-150): `LoadMessages`, `IdentitiesLoaded`, `CreateIdentity{identity, backed_up}`, `CreateContract{alias, contract_type, identity, inbox_state}`, `UpdateInboxPolicy`, `UpdateVerifiedBypass`, `CreateDelegate{aft_params}`, `CreateContact`, `FetchContactKeys`, `DeleteContact`, `RenameIdentity`, `DeleteIdentity`. `ContractType {InboxContract, AFTContract}` app.rs:152.
- `app()` (app.rs:167) provides `Signal<LoginController>`, `Signal<User>`, `Signal<InboxView>`, `InboxesData`, `AddressBookGen`, `ToastQueue` (also registered through `toast::set_root_queue`, app.rs:187) and the modal signals `login::ImportBackup`, `ImportContact`, `ShareContact`, `SharePending` (app.rs:197-200).
- One coroutine of `NodeAction`, two builds:
  - `use-node` without `no-sync` (app.rs:202-214): the receiver goes to `api::node_comms(rx, login_controller, user, inbox_data, ab_gen)`; app.rs handles nothing.
  - otherwise (app.rs:215-332): an in-page loop handles `CreateContact` / `DeleteContact`, `CreateIdentity` and `IdentitiesLoaded` (example-data only), `RenameIdentity`, `DeleteIdentity` (through `identity_store`, against the in-page delegate). All other actions are dropped (app.rs:327).
  - Both call `identity_store::set_action_sender(sync)`.
- No router. Screen choice at app.rs:374-400: `!user.identified` gives `login::GetOrCreateIdentity`; an open identity (`user.logged_id()`) gives `UserInbox`; else `login::IdentifiersList`. A change is logged through `LAST_BODY` (app.rs:352-372). Opening an identity is `user.set_logged_id(id)` plus `inbox.set_active_id(id)` (ui/src/app/login.rs:1192-1193, ui/src/app/settings.rs:161-162).
- Side effects in render: every render of `app()` with an identity open sends `NodeAction::LoadMessages` (use-node, app.rs:379-385; ui/src/api.rs:1973 answers with an inbox GET) and calls `load_example_messages` (example-data, app.rs:386-392).
- Storybook: example-data and `?story=<name>` render `storybook::Storybook` instead (app.rs:343-350).

### 2.2 View model (app.rs:408-1134, 1532-1570)

- `InboxesData(Signal<Vec<Rc<RefCell<InboxModel>>>>)` app.rs:413: v1 inbox models, written by api.rs, in arrival order (a position is not an owner, app.rs:615-621).
- `AddressBookGen(Signal<u32>)` app.rs:424.
- `UserId(usize)` app.rs:445: a handle, never an index; `NEXT_ID` starts at 16, above the example ids 0 and 1.
- `InboxView {active_id, messages}` app.rs:437: `messages` is the one list the sidebar, list and detail read.
- `User {logged, identified, active_id, identities}` app.rs:965; `logged_id()` and `set_logged_id()` look up by id (app.rs:1040-1049); `logout()` also clears the v2 outcome (app.rs:1053).
- `Message` app.rs:1061-1095: `id`, `from`, `title`, `content`, `read`, `time`, `sender_vk`, `signature_valid`, `assignment_hash`, `thread_id`, `in_reply_to`, `quoted_excerpt`. Equality and ordering are by `id` only (app.rs:1554-1570). `verification_state()` app.rs:1104; `stable_ref()` (bs58 of a non-zero `assignment_hash`) app.rs:1130.
- Four sources of a `Message`: `From<MessageModel>` (live v1, app.rs:1532), `kept_to_message` (delegate kept copy; zero hash, no thread fields, app.rs:1287), `v2_message` (app.rs:1434), `sent_message_to_row` (folded Sent, app.rs:3337).
- v1 ids are content hashes, not positions: `local_state::message_id_for` takes 8 bytes of blake3(ciphertext) and clears the top bit (ui/src/local_state/items.rs:70-92, ui/src/inbox.rs:1483-1487). Ids at or above `RECEIVED_ID_LIMIT = 1 << 63` (ui/src/local_state.rs:224) are reserved for rows that are not received mail.

### 2.3 Pure helpers (app.rs:1136-1530, 2883-2924, 3164-3374)

- Comparators, newest first with a total tie-break: `sort_cmp_inbox` (time, `assignment_hash`) app.rs:1155; `sort_cmp_drafts`, `sort_cmp_sent`, `sort_cmp_archive` app.rs:1163-1188.
- `merge_inbox_list(live, kept, is_archived, is_read, is_unread_import)` app.rs:1320: live rows minus archived with the read overlay; then kept rows not present by id and not archived, read unless an unread import.
- v2 rows: `is_v2_row(id)` is `v2::owner::row(id).is_some()` (app.rs:1367); `v2_rows` reads `v2::owner::inbound(addr, false)` (app.rs:1373); `v2_message` takes the id from `v2::owner::row_id`, the name and warnings from `v2::contacts::sender_name`, sets `signature_valid: true` and `assignment_hash: m.id`, and fills the `V2_SENDERS` thread-local (app.rs:1389) read by `v2_sender` and `V2SenderNotes` (app.rs:1396-1432). `merge_v2_rows` replaces every v2 row (app.rs:1527).
- Draft route: `ROUTE_DRAFT` / `draft_route` (app.rs:1466-1524): one `hashchange` listener per page; once the identity's drafts hold the id named by `#/compose?draft=<id>` (`ghostkeys::draft_in_route`), the hash is cleared and the draft opens in compose (the way back from getting a Ghost Key).
- `format_time_short` / `format_time_full` app.rs:2883-2924 (local time).
- `is_sender_verified` app.rs:3173; `is_inbox_row_visible` app.rs:3188; `folder_count` app.rs:3198 (Inbox and Quarantine count unread visible rows, the others totals); `matches_search` app.rs:3366.
- `visible_inbox_messages(emails, alias, folder, search)` app.rs:3237: drops archived, deleted, non-matching, unsigned (if `hide_unsigned`), applies the Inbox/Quarantine split when `quarantine_unknown` is on, and for Inbox only adds `folded_sent_rows` (app.rs:3301): the identity's Sent rows whose `thread_id` matches a visible received row or, without one, whose normalized subject does. Their ids are `SENT_FOLD_ID_BASE | fnv1a(sent uuid)` with `SENT_FOLD_ID_BASE = RECEIVED_ID_LIMIT` (app.rs:3331-3348).

### 2.4 Threads (`mod threads`, app.rs:1586-1874)

`MAX_THREAD_DEPTH = 4` (app.rs:1593). `ThreadGroup {key, messages (oldest first), root_id, unread_count, count}` (app.rs:1599). `normalize_subject` strips repeated `re:` / `fwd:` / `fw:` (app.rs:1620). `heuristic_key` is `heuristic:<subject>`, or `heuristic-blank:<id>` for an empty subject (app.rs:1664). `group_into_threads` (app.rs:1685): bucket by `thread_id` else heuristic key; fold each `heuristic:` bucket into the smallest-keyed `thread_id` bucket holding the same normalized subject; sort members by time then id; root is the oldest. Output order is HashMap order, so callers sort with `sort_cmp_group` (newest member, then `root_id` descending, app.rs:1795). `group_flat` (threading off) makes one group per message (app.rs:1771). `depth_map` (app.rs:1826) resolves `in_reply_to` through either the numeric id string or `stable_ref()`, ignores self links, bounds the walk by group size, caps at 4.

### 2.5 Selection model and shell (app.rs:2926-3736)

- `mod menu` (app.rs:2926-3162): `Folder {Inbox, Quarantine, Sent, Drafts, Archive}`; `ComposePrefill {to, subject, body, draft_id, thread_id, in_reply_to, quoted_excerpt}`; `SettingsScreen` (8 screens, `slug()` for `data-screen`); `MenuSelection {folder, email, sent_id, archived_id, thread_root, new_msg, search, compose_prefill, settings}`. `at_new_msg()` is a toggle (app.rs:3046); `set_folder` clears selections only on a change (app.rs:3130); `open_settings` closes compose (app.rs:3152).
- `UserInbox` (app.rs:3377): provides `Signal<MenuSelection>` and the drawer `Signal<bool>`; starts `v2::owner::start(addr)` for every confirmed identity, not only the open one (app.rs:3391-3395); renders `.fm-app` with `data-theme`, `data-density`, `data-font-size`, `data-view` (list or detail), then `Topbar`, `Sidebar`, `MessageList`, `DetailPanel`, `settings::SettingsShell()` (called as a function, app.rs:3466), `ComposeSheet` when `is_new_msg()`, `ToastView`, and the share and import modals.
- `Topbar` app.rs:3515; `brand_logo` app.rs:3489 (inlined SVG). `Sidebar` app.rs:3580: folder buttons carry `fm-folder-<label>` built by `format!` (app.rs:3651), Quarantine only when the setting is on; the connection card prints the literal `live` (app.rs:3691), the three-word key fingerprint and `app_version()`; Log out clears `active_id`, the v2 outcome and the message cache (app.rs:3717-3727).

### 2.6 List and detail (app.rs:3738-5199, 5619-5852)

- `MessageList` (app.rs:3739): on every render rebuilds `InboxView.messages` from the active identity's `InboxModel` plus `local_state::kept_for` through `merge_inbox_list` (app.rs:3749-3776), then replaces the v2 rows (app.rs:3779-3784); drains `DELAYED_ACTIONS` (app.rs:3787). Inbox and Quarantine render one `fm-thread-group` row per group (`data-msg-id` = root id, `data-thread-count`, `data-msg-card-testid`, app.rs:4140-4147), grouped by `group_into_threads` or `group_flat` per `global_settings().inbox.threading_enabled` (app.rs:4091-4098). Archive, Sent and Drafts render `msg-card` rows from `local_state::{archived_for, sent_for, drafts_for}`. v2 notices (`v2_view::ForeignNodeWarning`, `QueueNotice`, `LastOutcome`) sit above the list (app.rs:3882-3886); `data-v2-stored` exposes the delegate's drained count (app.rs:3891).
- `DetailPanel` (app.rs:4185): for Inbox and Quarantine, re-groups `visible_inbox_messages(.., "")` and finds the group by `root_id` (app.rs:4210-4219); Archive gives `OpenArchivedMessage`, Sent gives `OpenSentMessage`.
- `ThreadDetail` (app.rs:4399): marks every unread member read during render (`local_mark_read`, and `persist_read` under use-node, app.rs:4415-4449); picks `ThreadNested` (app.rs:4674) or `ThreadCompact` (app.rs:4795) from `inbox.thread_view`; Reply continues from the newest member. `ThreadRowActions` (app.rs:4548): Reply, Add to address book (verified-unknown senders), Archive, Delete; folded Sent rows get Reply only.
- `thread_reply_prefill` (app.rs:4340): To is the contact's local alias found by `sender_vk`, only when `signature_valid`; else, for a v2 row, the sender's address; else the display name. `thread_id` is the parent's or new; `in_reply_to` is `stable_ref()` or the id.
- `OpenSentMessage` (app.rs:5021): Reply (same thread, no `in_reply_to`), Forward and Resend (new conversation), Delete. `OpenArchivedMessage` (app.rs:4885): Reply by display name, Delete. `OpenMessage` (app.rs:5620) is unreachable (section 9).

### 2.7 Archive, delete (app.rs:5201-5617)

- `archive_message` (app.rs:5257). v2 row: local archive with `from = v2_reply_address(msg)`, then `v2::owner::archive_row(id, local_state::archive_message(..))` in a detached task; undone if the copy was not stored. v1 under use-node: `InboxView::defer_removal` binds the inbox by owner now, and `confirm_then_remove` removes from the contract only after the archived copy is stored (`store_then`, app.rs:5470), undoing otherwise.
- `delete_message` (app.rs:5391). v2: `delete_v2_message` then `v2::owner::delete_row`. v1: `local_delete_message`, `confirm_when_stored(local_state::delete_message)`, then `InboxView::remove_messages` queued on `DELAYED_ACTIONS`.
- `Confirmation`, `CONFIRM_DELETE`, `CONFIRM_ARCHIVE` (app.rs:5440-5465): the pending, done, failed and unconfirmed texts. `PostMessageAction` (app.rs:5212) stays in the thread unless it emptied or lost its root.
- `add_sender_to_address_book` (app.rs:5564): a v2 sender through `v2::contacts::add_sender`, a v1 one through the import modal.

### 2.8 Compose and send (app.rs:5854-6905)

`ComposeSheet` (app.rs:5895) takes the `ComposePrefill`, resolves the recipient chosen for it (`address_book::initial_choice`, `local_state::draft_recipient`) and keeps `to`, `chosen_address`, `ambiguous_draft`, `title`, `content`, `draft_id`, and the three thread fields as signals.

- Autosave on every input (app.rs:6008): `local_save_draft` and `set_draft_recipient` at once; under use-node a save debounced by `AUTOSAVE_DEBOUNCE_MS = 500` (app.rs:6001) calls `local_state::save_draft`, with an `OnCancel` flush if the sheet closes first and `pending_saves::begin()` counting it. `data-autosave` is `pending`, `saved` or `failed` (`autosave_state`, app.rs:5882).
- Click: `send_msg` (app.rs:6537) refuses a draft gone elsewhere, takes `claim_send`, and under use-node awaits `local_state::check_draft_before_send` before `send_now`; offline it calls `send_now` directly.
- `send_now` (app.rs:6164), in order:
  1. `migration_view::gate`: an unconfirmed migrated identity gets `ConfirmIdentityPrompt` first.
  2. An invite in To is refused (app.rs:6175).
  3. v2 when `v2::contacts::recipient_addr(&to)` is `Some` (an address in any form, or a v2 contact's name; ui/src/v2/contacts.rs:148): `v2::send::check_recipient`, a `v2::MailBody {v: 1, label, title, content, time_ms, thread_id, in_reply_to, quoted_excerpt}`, then `v2::send::send_and_record(sender_addr, alias, to_addr, to_name, to_val, body, ghost_for(GHOST_CHOICE, &to_addr))`, which writes the Sent row and its delivery state itself (ui/src/v2/send.rs:437). The draft is deleted and the sheet closed at once.
  4. v1 otherwise: `address_book::resolve_for_send`, the `verify_on_send` gate, `inbox::inbox_key_for_with_hash(vk, recipient.inbox_wasm_hash or INBOX_CODE_HASH)` (app.rs:6305), tier and max age from `contact_tier_cache::lookup` else the card (app.rs:6322), then `InboxView::send_message` (app.rs:479), whose future calls `InboxModel::sender_identity` and `DecryptedMessage::start_sending(client, ek, vk, tier, max_age, hash, &id)` (AFT token and inbox UPDATE, in inbox.rs / aft.rs) and on failure drops the pending ack and marks the Sent row Failed (app.rs:544-565). The Sent row is saved Pending (`local_save_sent`, `enqueue_pending_sent_ack`, `save_sent`, app.rs:6432-6506); api.rs flips it on the UpdateResponse.
- `with_signature` appends the identity's signature (app.rs:5867). `compose_post_send` gates draft removal, navigation and the "Sent" toast on success (app.rs:1261).

### 2.9 Example-data mode

`User::new` asks the in-page delegate for `address1` (UserId 0) and `address2` (UserId 1) and seeds the contact `alice-example` with placeholder keys (app.rs:973-1026). `load_example_messages` (app.rs:725) seeds five rows for address1 (a three-message thread `example-thread-0001`) and four for address2 (a legacy "Picnic" exchange), saves one Sent row for address2, and appends mail from `MOCK_SENT_MESSAGES` with ids `1_000_000 + UserId * 10_000 + i` (app.rs:917). A v1 send delivers into that map by recipient key (app.rs:574-605); the Sent row starts Delivered (app.rs:6436); contract removals are no-ops (app.rs:680-685). No v1 `InboxModel` is ever loaded offline. v2 works against the offline model in `v2::net`. `storybook` (app.rs:1884-2191) has stories `deep-nest`, `nested-vs-compact` and a default; `?view=compact` selects Compact.

### 2.10 testid.rs, branding, CSS

- `ui/src/testid.rs`: 164 `FM_*` constants (ui/src/testid.rs:18-237), each paired with a key of `ui/branding/testids.json` (164 keys). `ui/branding/app.json` holds `{"name": "Mail"}`, read by `branding::app_name()` (ui/src/lib.rs:53-73) and by `ui/tests/app-name.ts`.
- `ui/index.html` loads `vendor/css/fonts.css` (Fraunces, DM Sans, Geist Mono, local woff2), declares `@layer tokens, components`, then twelve partials, `responsive.css` last (ui/index.html:14-36). Mail partials are `@scope (.fm-app)`; `login.css` is `@scope (.fm-pre)`. Mobile is one breakpoint at 768px, driven by `data-view` and `.drawer-open` (ui/public/vendor/css/components/responsive.css:50-62). Font size is `zoom` on `data-font-size` (ui/public/vendor/css/components/main.css:15-18).

## 3. Interfaces

- Calls out: `api::{node_comms, WEB_API_SENDER, WebApiRequestClient, node_response_error_handling, TryNodeAction}`; `inbox::{InboxModel, MessageModel, DecryptedMessage, VerificationState, inbox_key_for_with_hash, INBOX_CODE_HASH, enqueue_pending_sent_ack, remove_pending_sent_ack, inbox_address_bs58_from_vk_bytes}`; `aft::{NO_TOKEN_RECORD_MSG_PREFIX, format_no_record_toast}`; `contact_tier_cache::lookup`; `local_state` (settings, drafts, sent, archived, kept, read and deleted marks, `GENERATION`, `Stored`, `DraftSave`, `DraftCheck`, id helpers); `pending_saves::begin`; `v2::{owner, contacts, send, MailBody}`; `ghostkeys::draft_in_route`; `address::{display, normalize_label, full_address, parse_recipient}`; `mail_delegate::ops` (example identities only); `toast`.
- Called by: ui/src/api.rs (`NodeAction`, `InboxesData`, `AddressBookGen`, `User`, `ContractType`), `app/settings.rs` and `app/login.rs` (`menu`, `User`, `InboxView`, `brand_logo`), the identity and v2 modules (`User`, `UserId`), and the tests of ui/src/local_state/items.rs (`store_then`). `threads`, `OnCancel`, `claim_send` and the time formatters have no caller outside app.rs.
- Types on the wire or in storage that app.rs builds: `freenet_mail_wire_v1::local_state::{Draft, SentMessage, ArchivedMessage, KeptMessage, DeliveryState}`, `inbox::DecryptedMessage`, `v2::MailBody`. A shape change to any of these is another subsystem's frozen format.
- Features: `use-node` (default), `no-sync`, `example-data` (which turns on `local-delegate` and `test-ghost-master`, ui/Cargo.toml:99-122); `use-node` with `local-delegate` is a compile error (ui/src/lib.rs:16-20). `cfg(target_family = "wasm")` gates `web_sys` use.
- DOM contract with Playwright: the `data-testid` values, plus `data-msg-id`, `data-thread-count`, `data-thread-view`, `data-msg-card-testid`, `data-sent-id`, `data-delivery-state`, `data-draft-id`, `data-autosave`, `data-v2-stored`, `data-story`, and the ids `email-inbox-accessor-<root>` and `email-content-<id>`.

## 4. Invariants and where they are held

- An identity or inbox is found by id or owner, never by position: `User::logged_id`, `pick_inbox` (app.rs:658); tests `deferred_removal_tests` (app.rs:6908).
- Archive removes from the inbox only after the archived copy is stored: `store_then`; `archive_order_tests` (app.rs:7985).
- Reading does not remove from the contract: `mark_as_read` (app.rs:709); `merge_inbox_list_tests`.
- A reply trusts `sender_vk` for addressing only with a valid signature: app.rs:4357; `reply_prefill_ignores_unverified_sender_vk_289` (app.rs:2787).
- Every sort is total and independent of HashMap order: `inbox_sort_tests`, `folder_sort_tests`, `groups_with_identical_time_and_hash_sort_deterministically`, `legacy_fold_target_is_deterministic_across_shared_subject_threads`.
- Draft removal, navigation and the success toast happen only after a successful send; one send per sheet at a time: `compose_post_send_tests` (app.rs:6989).
- A write whose view is about to unmount uses `spawn_forever`, never `spawn` (#286): app.rs:5286, 5379, 5501, 5540, 5574, 6065, 6155, 6407, 6499. Not test-pinned here.
- Sidebar badge and list apply the same filters (#268): `is_inbox_row_visible`; `folder_count_tests`.
- Folded Sent ids never collide with received ids: `SENT_FOLD_ID_BASE`; `sent_fold_tests`.
- Rust constants and the JSON agree exactly: `testid::tests::json_matches_rust_consts` (ui/src/testid.rs:246).
- A v2 sender is never added as a v1 contact, and a v2 archive keeps a reachable address: app.rs:5570, 5277-5283; `v2_sender_tests` (app.rs:8020).

## 5. State

Shipped: this is the live UI of both builds; nothing here is frozen bytes. It writes the v1 wire types above, which are frozen elsewhere. Candidate: everything touching `crate::v2` rides the inbox v2 candidate (AGENTS.md "Inbox v2 in the UI"). Unfinished: section 10.

## 6. Tests

Unit tests are in app.rs and run on the host with no Dioxus runtime (pure helpers and `futures::executor::block_on`):

- `thread_grouping_tests` (app.rs:2194-2876, 20 tests): grouping by `thread_id`, time ordering, legacy subject folding, blank subjects, deterministic fold target, depth and cap, dangling parents, numeric versus `stable_ref` linkage, group sort, and four #289 reply-prefill cases.
- `deferred_removal_tests` (3), `compose_post_send_tests` (4), `folder_count_tests` (2), `time_format_tests` (6), `kept_render_time_tests` (3), `kept_to_message_tests` (2), `quarantine_tests` (2), `sent_fold_tests` (8), `inbox_sort_tests` (6), `merge_inbox_list_tests` (3, one `should_panic` tripwire at app.rs:7752), `folder_sort_tests` (9), `autosave_state_tests` (1), `archive_order_tests` (3), `v2_sender_tests` (3).
- `testid::tests::json_matches_rust_consts`.

Commands (from Makefile.toml:907-919): `cargo test --workspace` (use-node build) and `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` (offline build); `cargo make test` runs both. Several tests share thread-local address book and local-state snapshots and reset them first (`address_book::clear_for_test`, `local_state::replace_snapshot`).

Playwright, offline build (`cargo make test-ui-playwright`; see the tests-qa map): `threads.spec.ts`, `threads-storybook.spec.ts`, `compose-autocomplete.spec.ts`, `email-app.spec.ts`, `repro-106-107.spec.ts`, `inbox-v2.spec.ts`. No component of this file has a Rust render test.

## 7. Changing it safely

- No frozen bytes, golden file or candidate rebuild follows from an edit to app.rs, testid.rs, branding or CSS alone. Clippy must pass in both feature sets; many items are `cfg`-gated and `dead_code`-allowed only offline (ui/src/lib.rs:7-10).
- A new test id needs three edits: the constant, the pair in the test list, the JSON key (ui/src/testid.rs:9-13). `ui/tests/testids.ts` has a hand-written interface that is not checked.
- Renaming a `data-*` attribute, a CSS class the specs select (`.is-me`, `.depth-N`, `.ft-count`), a folder label (it is the folder test id) or an example seed (titles and counts are asserted in `threads.spec.ts` and `email-app.spec.ts`; `docs/qa/manual-test-inventory.md` names them) breaks Playwright only, and Playwright is not part of `cargo make test`.
- `Message` equality is by id: a component taking `Message` or `ThreadGroup` props does not re-render when only content, read state or the address book changed (the reason `ThreadRowActions` computes its prefill at click time, app.rs:4567-4576).
- Hooks: `draft_route` and `settings::SettingsShell()` are plain functions whose hooks run in `UserInbox`'s scope; the second is conditional. Add hooks to `UserInbox` before them.
- `MessageList` is the only writer of `InboxView.messages`; `Sidebar` and `DetailPanel` read what it left.
- Anything that must survive the view closing needs `spawn_forever` and a toast queue captured beforehand (app.rs:5526-5529).
- A change to what a send or a draft carries goes through `wire-v1` types (`golden_v1.txt`) or `v2::MailBody`: see the protocol-wire, ui-state and ui-v2 maps before touching them.
- CSS is plain files under `ui/public`, loaded in a fixed order by `ui/index.html`; a new partial must be added there, before `responsive.css`.

## 8. Disagreements between docs or comments and code

1. Many comments call `Message.id` a per-inbox enumerate index that shifts on reload (app.rs:1117-1120, 1145-1151, 1316-1319, 3297-3299, 700-707, 4381-4386). It is a hash of the ciphertext (ui/src/inbox.rs:1483-1487). The `should_panic` tripwire at app.rs:7745-7776 guards a renumbering that can no longer happen.
2. The `threads` module doc and the `Message.thread_id` doc say legacy grouping is by subject plus participant (app.rs:1572-1581, 1086-1087); `heuristic_key` uses the subject alone (app.rs:1664-1671). `docs/qa/manual-test-inventory.md:172` repeats it and names a test, `groups_legacy_by_heuristic_subject_and_participant`, that does not exist (the test is at app.rs:2341).
3. The storybook comment says production does not fold Sent into inbox threads (app.rs:1976-1978), and `ui/tests/live-node.spec.ts:955-970` says the same; `folded_sent_rows` does (app.rs:3270-3283).
4. `NodeAction` docs name the identity-management delegate for contacts and identity deletion (app.rs:112, 125, 141) and describe rename as delete plus create (app.rs:129-134); both builds rename in place through `identity_store::rename_identity` (app.rs:281, ui/src/api.rs:2491-2497).
5. `MenuSelection::open_email` is said to be kept for deep links and search results (app.rs:3099-3102); nothing calls it.
6. `ui/Dioxus.toml` and `ui/index.html` say the title comes from an `APP_NAME` constant in lib.rs; there is none, the name is `branding::app_name()` from `ui/branding/app.json`. ui/src/lib.rs:166 mentions a `NewMessageWindow` component that does not exist.
7. `ui/tests/testids.ts` claims to mirror the JSON; its interface lacks 15 keys (among them `fmMsgTime`, `fmQuarantineCard`, `fmSettingsBtn`, `fmThreadingEnabledToggle`) and lists `fmArchiveUnarchive`, which is in neither the JSON nor Rust.
8. Two doc comments sit on the wrong function: the "Shared Delete logic" text is attached to `v2_reply_address` (app.rs:5356-5364) and the autosave-state text to `with_signature` (app.rs:5860-5867).
9. `Message.assignment_hash` is documented as the AFT assignment hash (app.rs:1078-1084); for v2 rows it is the v2 message id (app.rs:1457).

## 9. Suspected bugs and weak spots

1. With threading off, a reply row cannot be opened (high). The list uses `group_flat`, so a row's `root_id` is the message's own id (app.rs:4093-4097), but `DetailPanel` always resolves with `group_into_threads` (app.rs:4217-4219). A row that is not the oldest of a multi-message conversation resolves to nothing ("Select a message"); the oldest opens the whole thread. In the address1 seed that is two of five rows. No automated test uses the toggle (`fmThreadingEnabledToggle` appears in no spec).
2. Panic after deleting an identity on a node build (medium-high). `MessageList` does `InboxModel::contract_identity(&key).unwrap()` for each model until it finds the active one (app.rs:3752-3756). Deletion removes the routing (`InboxModel::forget_identity`, ui/src/inbox.rs:1227) but nothing removes the model from `InboxesData`. If the deleted identity's model precedes the open one's, render panics. `inbox_owner` handles the same case with `None` (app.rs:652).
3. Nested leaf detection compares numeric ids only (medium-high). `has_child` holds `in_reply_to` strings and is tested against `m.id.to_string()` (app.rs:4678-4695), but replies to rows with a hash carry `stable_ref()` (app.rs:4387). On real mail every message counts as a leaf and opens expanded. Offline seeds use numeric links, so the specs pass.
4. Expansion state is seeded once per component instance (medium). `ThreadNested` and `ThreadCompact` seed `use_signal` from the first group (app.rs:4696, 4802) and `ThreadDetail` has no key, so switching threads in place probably reuses it: Nested then opens everything, Compact nothing.
5. Stale badge and detail on new v1 mail (medium). Only `MessageList` subscribes to `InboxesData`; `Sidebar` and `DetailPanel` read the RefCell it rewrites during render (app.rs:3585, 4198), and `spawn_decrypt` signals nothing else (ui/src/api.rs:1690-1697).
6. Dead screens (high). `User::new` sets `identified: true` in both builds (app.rs:1022, 1032) and the only other write sets it true again (ui/src/app/login.rs:1870), so `login::GetOrCreateIdentity` is never shown (app.rs:374-377). `OpenMessage` (app.rs:5620-5852) cannot render because `open_email` has no caller; its Reply still addresses by display name.
7. The v1 send error arm cannot see an AFT error (medium-high). The only synchronous failure of `send_message` is `sender_identity` (app.rs:522), so the `NO_TOKEN_RECORD_MSG_PREFIX` branch (app.rs:6417) never matches and the user is told to check Settings, AFT for an identity-routing problem. "Sent" is toasted before the token is requested.
8. Appearance settings without effect (high for the selector mismatch). The density and serif-subject rules target `.msg-list-row`, `.fm-msg-row`, `.msg-subject`, `.fm-msg-subject` (ui/public/vendor/css/components/tokens.css:190-200); no Rust source emits those classes.
9. `delete_message` unwraps `remove_messages` (app.rs:5424-5427), which fails when the identity's inbox is not loaded (app.rs:690-692); the archive path logs the same error (app.rs:5329-5335) (low).
10. An inbox GET on most renders (medium): `app()` subscribes to `Signal<InboxView>` and sends `LoadMessages` each render; `mark_as_read` and removals write that signal.
11. Replies never nest under one's own messages (medium): folded Sent rows have a zero hash and a synthetic id, so an incoming reply's `in_reply_to` cannot resolve to them in `depth_map`.
12. A legacy Sent row with an empty subject can fold into the Inbox as its own row when any received subject is empty (app.rs:3313-3320 with app.rs:1666) (low).
13. The sidebar's connection state is the literal `live` (app.rs:3691), whatever the node connection is.

## 10. Open work

- Unarchive is hidden, blocked on an inbox contract `OwnerInsert` (#60) (app.rs:4962-4964).
- Kept copies do not store the assignment hash or thread fields (app.rs:1082-1084, 1303-1307).
- Two tabs can both send one draft inside a couple of delegate round trips; left open on purpose (app.rs:6529-6532).
- `MAIN_ELEMENT_ID` is unused, marked TODO (ui/src/lib.rs:46-47).
- The Rust/JSON test id pair is hand-synced "until we wire a build-time generator" (ui/src/testid.rs:11-13).
- Same-subject legacy conversations merge by design (app.rs:1651-1655).
- The #289 reply round trip on real nodes is deferred to #291 (app.rs:2728-2729).

## 11. Questions not answered from the tree

- Does Dioxus 0.7 reuse the `ThreadDetail` scope when another thread is opened in place (item 9.4)? A two-thread Playwright case would settle it.
- Is something outside app.rs meant to re-render `Sidebar` when v1 mail arrives (item 9.5)?
- Is the per-render `LoadMessages` the intended refresh, or a leftover?
- Should threading-off open a single message (the unreachable `OpenMessage`) or the whole thread?
- Are the density and serif settings meant to work, and against which class names?

---

## Appendix: the reader's structured summary

ui/src/app.rs (8,125 lines, about 5,000 non-test) is the Dioxus 0.7 root component and the whole signed-in mail UI: page contexts, the NodeAction enum the UI uses to reach the node loop, the in-memory inbox view model (InboxView, Message), thread grouping (mod threads), an example-data storybook, and the top bar, sidebar, message list, detail, thread and compose components. It owns no wire format or storage; it calls local_state, inbox/aft (v1) and v2. There is no router: app() picks login, identity list or inbox from the User signal, and a MenuSelection context picks folder, open thread, compose and settings. The list merges v1 contract rows, delegate kept copies, drained v2 rows and folded own-Sent rows into one Vec that MessageList rebuilds on every render. A send goes v2 when the To field resolves to a v2 address or v2 contact, else v1 through InboxView::send_message and DecryptedMessage::start_sending. testid.rs and ui/branding/testids.json hold 164 paired test ids, kept in step by a unit test; app.json holds the app name.

State: live code for both builds, nothing frozen here. Reading found several likely defects, none run: with threading off reply rows cannot be opened; a possible render panic after deleting an identity on a node build; nested leaf detection that only works for numeric links; density and serif settings whose CSS selectors match nothing; two unreachable screens. Many comments still describe message ids as positional though they are content hashes.

The map file is complete on disk, so a restart of this unit does not need to redo it.

### Key facts

- Screen switching has no router: app() renders login::GetOrCreateIdentity, UserInbox or login::IdentifiersList from User.identified and User.logged_id() (ui/src/app.rs:374-400); opening an identity is user.set_logged_id plus inbox.set_active_id (ui/src/app/login.rs:1192-1193).
- One NodeAction coroutine, two builds: use-node without no-sync hands the receiver to api::node_comms (ui/src/app.rs:202-214); otherwise an in-page loop handles only contact, identity create/rename/delete actions and drops the rest (ui/src/app.rs:215-332).
- Root contexts provided by app(): Signal<LoginController>, Signal<User>, Signal<InboxView>, InboxesData, AddressBookGen, ToastQueue and four modal signals (ui/src/app.rs:168-200); UserInbox adds Signal<MenuSelection> and the drawer Signal<bool> (ui/src/app.rs:3378, 3425-3426). The storybook must re-provide all of them (ui/src/app.rs:2140-2160).
- MessageList rebuilds InboxView.messages on every render from the active InboxModel plus kept copies (merge_inbox_list) and then replaces all v2 rows (ui/src/app.rs:3749-3785); Sidebar and DetailPanel read that RefCell and do not subscribe to InboxesData.
- Message.id for v1 rows is a hash of the ciphertext with the top bit cleared, not a position (ui/src/inbox.rs:1487, ui/src/local_state/items.rs:71-92); ids at or above RECEIVED_ID_LIMIT = 1<<63 are reserved for folded Sent rows (ui/src/local_state.rs:224, ui/src/app.rs:3331).
- Message equality and ordering compare id only (ui/src/app.rs:1554-1570), so components taking Message or ThreadGroup props do not re-render on content, read-state or address-book changes; ThreadRowActions computes its reply prefill at click time for this reason (ui/src/app.rs:4567-4576).
- Thread grouping: bucket by thread_id, else heuristic:<normalized subject> (subject alone), blank subjects stand alone; legacy buckets fold into the smallest-keyed thread_id bucket with the same subject (ui/src/app.rs:1664-1764). Depth comes from in_reply_to resolved by numeric id or stable_ref, capped at MAX_THREAD_DEPTH = 4 (ui/src/app.rs:1593, 1826-1873).
- Inbox display folds the identity's own Sent rows into visible conversations (thread_id match, else normalized subject) as synthetic rows; Quarantine does not fold (ui/src/app.rs:3270-3325).
- Send path choice is made in send_now: v2 when v2::contacts::recipient_addr(&to) is Some (an address in any form or a v2 contact's name), else v1 (ui/src/app.rs:6183-6240, ui/src/v2/contacts.rs:148). Before either: the migration confirm gate and the invite refusal (ui/src/app.rs:6168-6182).
- v2 send: v2::send::check_recipient, build v2::MailBody, then v2::send::send_and_record, which saves the Sent row and sets its delivery state itself (ui/src/app.rs:6189-6239, ui/src/v2/send.rs:437-507).
- v1 send: address_book::resolve_for_send, verify_on_send gate, inbox_key_for_with_hash with the contact's inbox_wasm_hash, tier from contact_tier_cache, then InboxView::send_message whose future calls DecryptedMessage::start_sending; the Sent row is saved Pending and acked through enqueue_pending_sent_ack (ui/src/app.rs:6244-6506, 479-608).
- Under use-node the Send click first awaits local_state::check_draft_before_send and holds claim_send; closing the sheet meanwhile cancels the send and toasts it via OnCancel (ui/src/app.rs:6537-6592).
- Compose autosave: local snapshot at once, delegate save debounced 500 ms (AUTOSAVE_DEBOUNCE_MS), flushed in a spawn_forever task if the sheet closes first; data-autosave exposes pending/saved/failed (ui/src/app.rs:6001-6108, 5882-5892).
- Archive of a v1 message removes it from the inbox contract only after the archived copy is stored, with the inbox bound by owner at click time (ui/src/app.rs:5309-5342, 5470-5485, 622-639); v2 archive and delete go through v2::owner::archive_row and delete_row (ui/src/app.rs:5277-5308, 5374-5389).
- Writes that must outlive the view use dioxus_core::spawn_forever (#286), at ui/src/app.rs:5286, 5379, 5501, 5540, 5574, 6065, 6155, 6407, 6499; a detached task must capture the toast queue beforehand (ui/src/app.rs:5526-5529).
- Example-data mode: User::new creates address1 (UserId 0) and address2 (UserId 1) in the in-page delegate and seeds contact alice-example (ui/src/app.rs:973-1026); load_example_messages seeds fixed rows that Playwright asserts on, and v1 sends land in the MOCK_SENT_MESSAGES thread-local (ui/src/app.rs:725-952, 574-605). No v1 InboxModel is loaded offline.
- Adding a test id needs three edits (const, pair in the test list, JSON key); testid::tests::json_matches_rust_consts enforces exact agreement of 164 entries (ui/src/testid.rs:246-443). Folder buttons use fm-folder-<label> built by format!, outside that list (ui/src/app.rs:3651).
- UserInbox starts the v2 inbox loop for every confirmed identity on the page, not only the open one (ui/src/app.rs:3391-3395), and app() sends NodeAction::LoadMessages on every render with an identity open (ui/src/app.rs:379-385).
- CSS is plain files loaded in fixed order by ui/index.html:14-36 (tokens, components, responsive.css last); mail rules are @scope (.fm-app), login rules @scope (.fm-pre); mobile layout keys off data-view and .drawer-open emitted by UserInbox (ui/src/app.rs:3427-3457).

### Docs or comments that disagree with the code

- **[medium]** Claim (code comments, ui/src/app.rs:1117-1120, 1145-1151, 1316-1319, 3297-3299, 700-707, 4381-4386, and the should_panic tripwire test at 7745-7776): Message.id is the per-inbox enumerate index of the contract's message vec and shifts on reload or eviction **Reality:** v1 ids are derived from a blake3 hash of the ciphertext (message_id_for), stable across reloads and reordering; the renumbering hazard the tripwire documents no longer exists Evidence: ui/src/inbox.rs:1483-1487; ui/src/local_state/items.rs:71-92
- **[low]** Claim (threads module doc ui/src/app.rs:1572-1581; Message.thread_id doc ui/src/app.rs:1086-1087; docs/qa/manual-test-inventory.md:172 (which also names a test groups_legacy_by_heuristic_subject_and_participant)): Legacy (no thread_id) mail is grouped by normalized subject plus participant **Reality:** heuristic_key uses the normalized subject alone (blank subjects per message); the named test does not exist, the real one is groups_legacy_by_heuristic_subject_across_senders Evidence: ui/src/app.rs:1664-1671; ui/src/app.rs:2341
- **[low]** Claim (storybook comment ui/src/app.rs:1976-1978; ui/tests/live-node.spec.ts:955-970): Production does not fold Sent mail into the inbox thread; the inbox Vec is the only input to group_into_threads **Reality:** visible_inbox_messages adds folded_sent_rows for the Inbox folder, so own Sent rows join matching conversations Evidence: ui/src/app.rs:3270-3283, 3301-3325
- **[low]** Claim (NodeAction doc comments ui/src/app.rs:112, 125, 129-134, 141): Contacts and identity deletion go through the identity-management delegate; RenameIdentity is implemented as DeleteIdentity plus CreateIdentity **Reality:** Both builds rename in place through identity_store::rename_identity against the mail delegate Evidence: ui/src/app.rs:278-293; ui/src/api.rs:2491-2497
- **[low]** Claim (comment ui/src/app.rs:3099-3102 and 4235-4236): open_email / the single-message OpenMessage path is retained for deep-link and search-result selection **Reality:** open_email has no caller anywhere, so MenuSelection.email is never set and OpenMessage never renders Evidence: ui/src/app.rs:3103-3109 (only definition); grep of ui/src finds no call
- **[low]** Claim (ui/Dioxus.toml (web.app comment) and ui/index.html:4-5): The page title comes from the APP_NAME constant in ui/src/lib.rs **Reality:** There is no APP_NAME constant; the name is branding::app_name(), read from ui/branding/app.json Evidence: ui/src/lib.rs:53-75
- **[low]** Claim (ui/tests/testids.ts header comment and its TestIds interface): ui/tests/testids.ts mirrors ui/branding/testids.json **Reality:** The interface has 150 keys against 164 in the JSON: 15 are missing (e.g. fmMsgTime, fmQuarantineCard, fmSettingsBtn, fmThreadingEnabledToggle) and fmArchiveUnarchive is in neither the JSON nor Rust Evidence: ui/tests/testids.ts:12-160; ui/branding/testids.json:1-172
- **[low]** Claim (ui/src/lib.rs:164-167): Offline WEB_API_SENDER stub exists so components like NewMessageWindow and OpenMessage do not panic **Reality:** No NewMessageWindow component exists; the compose component is ComposeSheet Evidence: ui/src/app.rs:5895
- **[low]** Claim (ui/src/app.rs:5356-5359 and 5860-5863): Doc comment 'Shared Delete logic ... Shared between OpenMessage and the threaded per-message action clusters' and the doc for the autosave state **Reality:** Both are attached to the wrong function: the first sits on v2_reply_address, the second on with_signature Evidence: ui/src/app.rs:5364; ui/src/app.rs:5867
- **[low]** Claim (ui/src/app.rs:1078-1084): Message.assignment_hash is the AFT TokenAssignment.assignment_hash **Reality:** For v2 rows it is the v2 message id; for kept and folded Sent rows it is zero Evidence: ui/src/app.rs:1457

### Suspected issues (leads from reading, not confirmed)

- **With threading disabled, reply rows cannot be opened** (confidence: high). The list builds one group per message with group_flat (root_id = the message's own id) but DetailPanel always resolves the open thread with group_into_threads and matches on root_id. A row that is not the oldest message of a multi-message conversation resolves to None and shows 'Select a message'; the oldest row opens the whole thread. No spec uses the threading toggle. Evidence: ui/src/app.rs:4091-4098 and ui/src/app.rs:4210-4232
- **Possible render panic after deleting an identity on a node build** (confidence: medium). MessageList calls InboxModel::contract_identity(&key).unwrap() on each loaded model until it finds the active identity's. Deleting an identity removes its routing entry (InboxModel::forget_identity) but nothing removes its model from InboxesData, so if that model precedes the open identity's in arrival order the unwrap panics. inbox_owner handles the same lookup with None. Evidence: ui/src/app.rs:3752-3756; ui/src/inbox.rs:1227; ui/src/api.rs:1458-1464 (only push, never remove)
- **Nested view leaf detection only works for numeric in_reply_to** (confidence: medium). has_child holds in_reply_to strings and is tested against m.id.to_string(), but replies to rows with a non-zero assignment_hash carry stable_ref (bs58). On real v1 and v2 mail no message is found to have a child, so every message is treated as a leaf and expanded. Offline seeds use numeric links, so Playwright passes. Evidence: ui/src/app.rs:4678-4695 against ui/src/app.rs:4387
- **Thread expansion state is seeded once per component instance** (confidence: medium). ThreadNested and ThreadCompact seed use_signal from the first group they render and ThreadDetail carries no key, so opening another thread in place likely reuses the state: Nested falls back to open for every message, Compact to closed for every message including the newest. Evidence: ui/src/app.rs:4691-4697, 4716; ui/src/app.rs:4797-4802, 4811; ui/src/app.rs:4221
- **Sidebar badge and open thread can lag behind newly arrived v1 mail** (confidence: medium). Only MessageList subscribes to InboxesData; it rewrites InboxView.messages (a RefCell) during render without notifying anyone. Sidebar and DetailPanel read that RefCell and have no subscription that fires when api.rs adds decrypted messages. Evidence: ui/src/app.rs:3585, 4198-4199, 3751; ui/src/api.rs:1690-1697
- **Two unreachable screens** (confidence: high). User.identified is true from construction in both builds and is only ever set to true again, so the GetOrCreateIdentity branch of app() never renders. OpenMessage (about 230 lines) cannot render because open_email has no caller; its Reply still addresses by display name rather than by the #289 VK lookup. Evidence: ui/src/app.rs:1022, 1032, 374-377; ui/src/app/login.rs:1870; ui/src/app.rs:3103-3109, 5795-5803
- **v1 send error arm cannot see an AFT failure** (confidence: medium). The only synchronous failure of InboxView::send_message is InboxModel::sender_identity, so the NO_TOKEN_RECORD_MSG_PREFIX branch never matches and an identity-routing error is shown as 'Check Settings, AFT'. The 'Sent' toast and draft deletion happen before the token is requested; real failures arrive later through node_response_error_handling. Evidence: ui/src/app.rs:522, 6414-6424, 6513-6522
- **Density and serif-subject appearance settings match no element** (confidence: high). The CSS rules for data-density=compact and the serif-subjects opt-out select .msg-list-row, .fm-msg-row, .msg-subject and .fm-msg-subject; no Rust source emits those classes (the list uses .msg-card, .msg-subj, .ft-thread-group, .ft-group-subj). Evidence: ui/public/vendor/css/components/tokens.css:190-200; ui/src/app.rs:3405-3419, 4136-4168
- **delete_message unwraps a removal that can fail** (confidence: low). InboxView::remove_messages returns Err when the identity's inbox is not loaded; delete_message unwraps it while the archive path logs the same error. Evidence: ui/src/app.rs:5424-5427; ui/src/app.rs:690-692; ui/src/app.rs:5329-5335
- **Inbox GET on most renders of app()** (confidence: medium). app() reads Signal<InboxView> and sends NodeAction::LoadMessages on each render with an identity open; mark_as_read and removals write that signal, so opening an unread thread or deleting a message triggers another GET and set_contract_identity. Evidence: ui/src/app.rs:379-385, 4422-4425; ui/src/api.rs:1973-1986
- **Replies never nest under the user's own messages** (confidence: medium). Folded Sent rows have a zero assignment_hash and a synthetic id, so an incoming reply whose in_reply_to names the user's sent message cannot resolve in depth_map and renders at depth 0. Evidence: ui/src/app.rs:3348-3362; ui/src/app.rs:1831-1853
- **Connection status is a literal** (confidence: high). The sidebar connection card always prints 'live', independent of the node connection. Evidence: ui/src/app.rs:3685-3692
- **Blank-subject legacy Sent rows can surface as standalone Inbox rows** (confidence: low). folded_sent_rows matches a thread-id-less Sent row on normalized subject, including the empty string, but group_into_threads gives blank subjects a per-message key, so such a row appears alone in the Inbox. Evidence: ui/src/app.rs:3313-3320; ui/src/app.rs:1665-1667

### Open work

- Unarchive is hidden, blocked on an inbox contract OwnerInsert (#60): ui/src/app.rs:4962-4964
- Kept copies do not persist assignment_hash or thread fields, so kept rows group only by subject and link by numeric id: ui/src/app.rs:1082-1084, 1303-1307
- Two tabs can both send the same draft within a couple of delegate round trips; left open deliberately: ui/src/app.rs:6529-6532
- MAIN_ELEMENT_ID unused, 'TODO: wire into the Dioxus mount point': ui/src/lib.rs:46-47
- Rust/JSON test id pair is hand-synced 'until we wire a build-time generator': ui/src/testid.rs:11-13
- Same-subject legacy conversations merge into one group by design (documented tradeoff): ui/src/app.rs:1651-1655
- Full UI reply round trip for #289 on real nodes deferred to #291: ui/src/app.rs:2728-2729
- mark_as_read returns a no-op future still routed through DELAYED_ACTIONS (vestigial): ui/src/app.rs:709-722, 3787-3792
- Threading on/off toggle and Quarantine folder have manual coverage only: docs/qa/manual-test-inventory.md:153, 166

### Open questions

- Does Dioxus 0.7 reuse the ThreadDetail / ThreadNested scope when a different thread is opened in place? That decides whether the stale expansion state (suspected issue 4) is real; a two-thread Playwright case would settle it.
- Is anything outside app.rs meant to re-render Sidebar and DetailPanel when v1 mail arrives, or is the unread badge expected to lag until the next unrelated render?
- Is sending NodeAction::LoadMessages on every render of app() the intended refresh mechanism or a leftover?
- With threading off, should a row open a single message (the currently unreachable OpenMessage) or the whole conversation?
- Are the density and serif-subject settings meant to work, and against which class names?
- Can a deleted identity's InboxModel really precede the open identity's in InboxesData on a live node (suspected issue 2)? It needs a node run with two identities to confirm.
- docs/codebase-notes/ already exists inside the repository with six area notes although the fork is described as unchanged; it has no ui-app-shell note. Whether this map should be added there is the lead's call.

### Coverage

Read in full: `ui/src/testid.rs`, `ui/branding/app.json`, `ui/branding/testids.json`, `ui/public/vendor/css/fonts.css`, `ui/public/vendor/css/components/main.css`, `ui/src/lib.rs`, `ui/index.html`, `ui/tests/testids.ts`, `ui/tests/app-name.ts`, `ui/src/app.rs (all non-test code, lines 1-2192 and 2877-6905, in full; test modules: every test name, and bodies of thread_grouping_tests, deferred_removal_tests, compose_post_send_tests, folder_count_tests, quarantine_tests, merge_inbox_list_tests, autosave_state_tests, archive_order_tests, v2_sender_tests)`

Skimmed: `ui/src/app.rs test modules time_format_tests, kept_render_time_tests, kept_to_message_tests, sent_fold_tests, inbox_sort_tests, folder_sort_tests (names and helper signatures only)`, `ui/public/vendor/css/components/{tokens,topbar,sidebar,message-list,detail,thread,compose,toast,login,settings,responsive}.css (headers, scopes, selectors that app.rs drives)`, `ui/src/v2/owner.rs (start, row, row_id, archive_row, delete_row, inbound)`, `ui/src/v2/send.rs (check_recipient, send_and_record, GHOST_CHOICE)`, `ui/src/v2/contacts.rs (recipient_addr, is_invite, by_addr, GEN)`, `ui/src/local_state.rs and ui/src/local_state/items.rs (id helpers, RECEIVED_ID_LIMIT, should_persist_read, check_draft_before_send)`, `ui/src/inbox.rs (contract_identity, forget_identity, load, message id assignment)`, `ui/src/api.rs (LoadMessages, RenameIdentity, DeleteIdentity handlers, spawn_decrypt, forget_inbox_states)`, `ui/src/app/settings.rs (SettingsShell head)`, `ui/src/app/identity_store.rs (drop_from_page, forget_everywhere)`, `ui/Dioxus.toml`, `ui/Cargo.toml (features)`, `Makefile.toml (test tasks)`, `ui/tests/live-node.spec.ts (lines 950-975)`, `ui/tests/threads.spec.ts, threads-storybook.spec.ts, compose-autocomplete.spec.ts (test names)`, `docs/qa/manual-test-inventory.md (threading rows)`, `docs/codebase-notes/README.md (head)`

Skipped: `None under the owned paths. ui/public/vendor/css partials other than fonts.css and main.css were skimmed, not read line by line, as the task instructed.`
