> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# ui-app-screens: login and identities, address book, settings, backup, migration status

Paths: `ui/src/app/{login,address_book,settings,backup_token,backup_panel,migration_view,migration_fixture}.rs`, `ui/src/app/testdata/`. Line references are repo-relative; a bare number refers to the file named in the heading above it. Nothing was built or run.

## 1. Purpose and place in the system

The Dioxus screens outside the mailbox, plus two non-UI models the rest of the UI leans on:

- `login.rs`: the page's `Identity` type and live identity list (`ALIASES`), the first-run and "Your identities" hub, the create, restore, rename, delete and backup flows, and the share, import and verify contact modals.
- `address_book.rs`: the in-memory alias → entry map, the six-word fingerprint, the contact card codec and recipient resolution for sending. No UI.
- `settings.rs`: the full-screen Settings shell (eight screens).
- `backup_token.rs`, `backup_panel.rs`: the recovery-key codec wrapper and the panel that shows, copies and downloads a key.
- `migration_view.rs`: the page side of the v0.2.x migration (the walk is `ui/src/migration/`, sibling map `ui-migration`); `migration_fixture.rs` scripts it for offline Playwright.

Private keys are in the mail delegate (AGENTS.md "The mail delegate", correct); the page holds an identity's delegate address and public keys (login.rs:56-84). Routing: the root shows `GetOrCreateIdentity` when `!user.identified`, `UserInbox` when an identity is open, else `IdentifiersList` (ui/src/app.rs:374-399). Settings renders inside `UserInbox` (app.rs:3465-3467).

## 2. Walkthrough

### ui/src/app/login.rs (3298 lines)

**Identity model.** `Identity {alias, id: UserId, description, required_tier, max_age_secs, addr, confirmed, generation, ek_bytes(1184), vk_bytes(1952)}` (59-84). `from_info` builds it from the delegate's `IdentityInfo` and hard-codes `Tier::Min10` and `DEFAULT_MAX_AGE_SECS` (88-101). `PartialEq`/`Hash` use the ML-DSA vk bytes only (117-122, 403-408), so a component taking an `Identity` prop does not re-render on rename; callers pass `alias_str` separately (973-980, 1432-1433).

`ALIASES` is a thread-local `Vec<Identity>` (50-54), not a signal. Writers: `upsert` (160-199: replace by `addr`, keep the held entry if its `generation` is newer, reuse the slot's `UserId`), `set_identities` (138-151, startup snapshot, #286), `set_alias` (203-215, also routes the inbox key and registers in the address book), `rename_in_place` (361-374), `remove_in_place_by_addr` (393-400), `set_confirmed`/`set_generation`/`set_page_alias` (298-356). Readers: `get_aliases` (sorted copy, 217-223), `get_alias` (226-232), `by_addr` (293-295). Re-render is forced by `LoginController.updated` (898-907).

**Legacy backup types.** `StoredIdentityKeys {ml_dsa_seed: Vec<u8> (32), ml_kem_seed: Vec<u8> (64)}`, serde JSON arrays of integers, zeroized on drop (427-449). `IDENTITY_BACKUP_VERSION = 2` (488). `IdentityBackup {version, alias, description, keys}` (497-503) is both the old file format and the in-memory result of any restore parse.

**Backup stamp.** `mark_backed_up(alias)` sets `IdentitySettings.last_backup_at = now` (521-525), called only from "I've saved it". `record_backup_after_create(alias, backed_up)` (543-552) is called by the node loop when it handles `NodeAction::CreateIdentity` (ui/src/api.rs:2017) and by the offline coroutine (app.rs:265).

**Downloads.** `backup_filename` → `freenet-identity-<sanitized alias>.txt`, falling back to the first two fingerprint words (559-605). `DOWNLOAD_GAP_MS = 2_100` with a const assert `> 2_000` (636-638): the gateway shell drops a second download within 2 s, app-wide, so the gap is enforced once in `trigger_browser_download` (675-692) with a monotonic `performance.now()`. In an iframe it posts `{__freenet_shell__: true, type: "download", filename, mimeType, base64}` to the gateway origin, never `"*"` (707-757); at top level it clicks a local `<a download>` (762-797). Results: `DownloadRoute::{Local, Relayed}`, `DownloadError::{TooSoon, Failed}` (608-628). The host build is a stub returning `Ok(Local)` (799-806).

**Modal state (context signals).** `ImportBackup(bool)`, `ShareContact(bool)`, `SharePending{data: {alias, share_text}}`, `ImportContact{open, prefill, prefill_alias}` (808-866) are provided at the app root (app.rs:197-200); `CreateAlias`, `ImportId`, `VerifyContactPending` are provided by `IdentifiersList` (911-921).

**Hub (`Identities`, 962-1424).** Per row (`IdentityEntry`, 979-1281): avatar initial, alias, description, first two fingerprint words (990-996), and buttons Rename, Back up, Share, Delete, "Open inbox" (1141-1196).
- Rename (1039-1076): inline editor; refuses empty, an own alias in use, or a contact's name (`address_book::is_contact_name`, 1065); sends `NodeAction::RenameIdentity{old, new, identity}`.
- Delete (1078-1093, modal 1200-1255): the alias must be typed exactly; sends `NodeAction::DeleteIdentity`.
- Share (1168-1174): goes through `migration_view::gate(confirmed)`; an unconfirmed (migrated) identity gets `ConfirmIdentityPrompt` first (1256-1265). `open_share` builds `"{inbox address}\nverify: {w1-w2-w3-w4-w5-w6}"` (1021-1036).
- Back up: `BackupKeyModal` (1430-1489) wraps `ExportedKeyPanel`; it also opens when `migration_view::BACKUP_REQUEST` names this identity (1019-1020, 1266-1279). A backdrop click closes it only if the press started on the backdrop (1443-1452).
- Above the rows: Create and Restore buttons, hidden while the list is empty and the migration is walking (1288, 1357-1372); "Repair storage" when `IDENTITY_INDEX_NEEDS_REPAIR` (1327, 1373-1393); one "Repair" per `local_state::unreadable_lists()` entry (1332-1347, 1394-1414); "Finish deleting" per `UNFINISHED_DELETIONS` (1298-1323); then `MigrationPanel` (1416).

**Create (`CreateAliasForm`, 1492-1828).** Form stage: alias required and not an existing own alias (1532-1542). `ops::create_identity(alias, description)`; on `CallError::Unknown` it calls `ops::after_lost_create` and accepts only `LostCreate::Created` (1551-1562). Reveal stage holds `(Identity, SecretString, [&str; 6])` in a signal (1507-1512): six words, a nudge, and, only after "Back up now", `BackupKeyPanel` with the key returned at creation (1663-1668, 1723-1747). "Continue to inbox" calls `identity_store::publish_or_recover(identity, backup_confirmed)` (1582-1593). "Discard" deletes the identity in the delegate (which prompts) via `ops::delete_identity` (1605-1626), after a warning dialog if the key was copied, downloaded or a copy is in flight (1644-1653, 1671-1692). The Discard button exposes `data-key-exported` and `data-copies-in-flight` for tests (1756-1758).

**Restore (`ImportForm`, 1920-2312).** One textarea plus a file input (`.json,.txt`). Every edit runs `backup_token::parse_backup` (1949-2004); a typed (not pasted) edit that could still become valid holds its error until blur (1996-1999, 2065-2069). On success the page derives vk and ek from the seeds to show the "Recovered fingerprint" (1979-1991) and refuses keys already on the device by vk (2010-2026). Files over 64 KiB are not read (2096-2102). Submit requires an alias not in use (2227-2237), then sends `ops::import_identity(ImportSource::Seeds{dsa, kem}, ImportKind::UserRestore, alias, description)` (2240-2262). On success it zeroizes the paste text, and unless the delegate said `already_present` for a listed identity, runs `publish_or_recover(identity, true)` (2264-2297).

**Contacts.** `ShareContactModal` (2316-2431) splits the share text into the token and the `verify:` words, exposes `data-share-text`, and copies via `clipboard::copy_text`. `ImportContactForm` (2461-2928): `apply_address` (2567-2670) takes the first non-empty non-`verify:` line, `ContactCard::decode`s it and, under `use-node`, sends `NodeAction::FetchContactKeys{inbox_address}`; a `use_future` polls `api::contact_import::RESULTS` every 150 ms and gives up after 200 ticks, 30 s (2494-2561). Without `use-node` it synthesizes keys from the address with BLAKE3 unless `lookup_by_bs58` knows it (2616-2653). `FetchedKeys = (vk, ek, Tier, max_age_secs, Option<inbox_wasm_hash>)` (2452-2458). Import (2869-2921) refuses an own fingerprint, builds `StoredContactKeys` and sends `NodeAction::CreateContact{contact}`. The prefill effect serves "Add to address book" from a message (2677-2707, caller app.rs:5608). `persisted_suggested_alias` keeps the advertised name only when it differs from the local label (882-889). `VerifyContactModal` (2936-3029) re-sends `CreateContact` with `verified = true`. `ContactsSection` (3033-3145) lists contacts with verify and remove (`NodeAction::DeleteContact{alias}`, no confirmation).

### ui/src/app/address_book.rs (2199 lines, 1131 onward are tests)

- `fingerprint_words(vk, ek)` (34-56): BLAKE3 over `vk || ek`, first 12 bytes big-endian, six 11-bit indices into the BIP-39 English list (`bip39_english.txt`, 2048 words). 66 bits.
- `ContactCard {version, inbox_address, suggested_alias, suggested_description}`, `VERSION = 3` (67-87). `encode()` returns the bare bs58 inbox `ContractInstanceId` and nothing else (92-94); `decode` parses a bs58 instance id and always sets both suggestions to `None` (100-111). `from_identity` derives the address with `inbox::inbox_key_for(vk)` (113-138).
- `Contact {local_alias, suggested_alias, description, ml_dsa_vk_bytes, ml_kem_ek_bytes, required_tier, max_age_secs, verified, inbox_wasm_hash}` (143-168); `Recipient` adds `fingerprint` and `is_own` (196-215).
- State: `ADDRESS_BOOK: HashMap<String, Entry{Own|Contact}>`, `BS58_LOOKUP_CACHE` keyed by inbox instance id, `ADDRESS_BOOK_GEN`, `RELABELLED` (label → the alias lost) (234-266).
- Mutators: `register_identity` (487-517; a same-named contact moves to its clash label and the user gets a toast), `insert_contact` (526) and `load_contact` (536) over `insert` (554-618), `remove_contact` (644-674; the last clash-labelled contact gets the bare name back), `rename_own_identity` (680), `remove_own_identity` (716). `persist_after_removal(vk)` → `Delete | Rewrite(kept)` (625-635).
- Resolution: `lookup(alias)` (769-794: exact key, else the single contact whose `suggested_alias` matches and is not relabelled), `lookup_by_bs58` (870-887, cached), `address_choice` (320), `initial_choice` (333), `recipient_address` = `"fmail-addr:" + bs58(address_of(vk))` (352-361), `resolve_for_send(typed, chosen, ambiguous)` (418-462), `key_conflict` (542-552).
- Other readers: `is_contact_inbox_key` (895-919), `all_contacts` (935), `filter_contacts` (964-1004: alias prefix, alias substring, fingerprint; never description), `contact_by_vk` (1009), `is_own_fingerprint` (1034), `own_names` (922), `is_contact_name` (705).
- `StoredContactKeys {ml_dsa_vk_bytes, ml_kem_ek_bytes, suggested_alias, verified, required_tier (default Min10), max_age_secs (default 365 d), inbox_wasm_hash (omitted when None)}` (1050-1070); `into_contact` (1096).

### ui/src/app/settings.rs (1521 lines)

`SettingsShell` (63-200): back button, identity switcher popover, nav, and one screen. Per-identity screens are keyed by alias so a switch remounts them (186-195). `use_identity_settings` reads the alias inside the memo (37-52), `use_global_settings` (54-60).

| Screen | Lines | What it writes |
|---|---|---|
| Account | 372-616 | `signature`, `auto_sign`; recovery-key card (`ExportedKeyPanel`, 475-507) guarded by `backup_for` = first 16 vk bytes in hex (436-450); `mark_backed_up` on confirm (493); "Restore…" (465-468); "Revoke" → `ops::revoke_export_grant` (601-612, 1480-1493) |
| Privacy | 619-655 | `privacy.verify_on_send`, `privacy.hide_unsigned` |
| AFT | 765-1041 | `aft.required_tier`, `aft.max_age_days` (clamped 1..=730, 822) → `NodeAction::UpdateInboxPolicy` (790-808); `aft.allow_verified_skip_token` → `NodeAction::UpdateVerifiedBypass` with every verified contact's vk (836-864, 756-762); usage panel from `AftRecords::used_counts_by_tier`, `CAPACITY_AMPLE_THRESHOLD = 10_000` (689, 878-896) |
| Inbox | 1044-1114 | global `inbox.drafts_in_inbox`, `inbox.thread_view`, `inbox.quarantine_unknown` |
| Contacts | 1117-1269 | search, Import, "Share my card" (gated, 1153-1159), Remove |
| Address | 192 | `v2_view::ScrAddress` (sibling map `ui-v2`) |
| Appearance | 1272-1401 | `appearance.{theme, density, font_size, serif_subjects}`, `inbox.threading_enabled` |
| Advanced | 1420-1476 | `advanced.custom_relay`, `advanced.custom_relay_url` (read at api.rs:69-76 on the next load) |

Tier tables: `TIER_ROWS` (734-750), `ALL_TIERS` (695-701), `parse_tier` (661), `tier_display_name` (708). All of the toggles have consumers (app.rs:3188-3259, 3829, 5869, 6263; ui/src/v2/send.rs:427).

### ui/src/app/backup_token.rs (681 lines)

A wrapper over `freenet_mail_protocol::recovery_key` (protocol/src/recovery_key.rs:34-51): `fmail-key-1:` + base58 of `[0x01][dsa seed 32][kem seed 64][4-byte checksum]`, checksum = first 4 bytes of SHA-256 over prefix + body, `PAYLOAD_LEN = 101`, `MAX_TOKEN_CHARS = 512`. `encode` is `#[cfg(test)]` (46-62): the page never builds a key. `parse_backup(text)` (196-217): over `MAX_RESTORE_INPUT_BYTES = 64 KiB` → `TooLarge`; first non-blank char `{` → `parse_json_backup` (219-241: version probe first; only version 2; seed lengths checked), then `MAX_ALIAS_CHARS = 64` / `MAX_DESCRIPTION_CHARS = 1024` (186-187, 203-207); otherwise `decode`. A key yields empty alias and description. Helpers: `may_be_incomplete` (79-94), `is_typed_edit` (102-114), `names_to_keep` (120-122). `BackupParseError` (126-180).

### ui/src/app/backup_panel.rs (639 lines)

`ExportedKeyPanel{addr, alias, words, saved, on_saved, scroll_request}` (185-222) calls `ops::export_recovery_key(addr)` in a `use_resource`, showing "Approve showing the recovery key in the prompt from your Freenet node" until it answers. `BackupKeyPanel` (225-510): suggested entry title `Freenet Mail: <alias> (<w1> <w2>)` (95-103); key masked as prefix + 28 dots until "Show key" (107-114, 414-435); Copy (283-340); "Download as .txt" writes `"<key>\n"` as `text/plain` (342-370) with a 2 100 ms button cooldown; "I've saved it" fires `on_saved`. `CopyTracker {generation, in_flight, landed}` (39-90) is owned by the create flow so a copy that settles after the panel unmounted still counts; `classify_tracker_write` and `copy_report` (136-178) decide `Full | StatusOnly | Nothing`.

### ui/src/app/migration_view.rs (444 lines)

`MIGRATION: GlobalSignal<MigrationView>` (24-41, 66) with `walking, clearing, unreachable, totals, migrated, rejected, in_progress, notice_dismissed, orphans, report`. `BACKUP_REQUEST: GlobalSignal<Option<Addr>>` (64). `gate(confirmed: Option<bool>) -> Gate::{Ask, Go}`: `Ask` only for `Some(false)` (46-61). `notice_summary` (70-89). `after_listing(transport, inbox)` (129-171) is the whole post-listing sequence: `migration::run` → reload identities if anything arrived → set `walking=false`, `unreachable`, `in_progress` → `refresh` → `migration::scrub::run` (sets `clearing`) → `migration::confirm_by_inbox` → `refresh`. `refresh` (96-125) reads the ledger and stored orphans and logs `migration status: <report>`. `MigrationPanel` (303-379) renders the status line, the unreachable note with a Restore button, the notice (summary, `RETRY_DISCLOSURE`, one "Save the recovery key for X" per migrated identity, Dismiss), orphan rows (move to a chosen identity, or discard after a listing, 223-300) and a collapsible JSON report. `ConfirmIdentityPrompt` (383-444) shows address and fingerprint and calls `identity_store::confirm_identity`.

### ui/src/app/migration_fixture.rs (97 lines) and testdata/

`example-data` + wasm only (app.rs:335-336). `?migration-fixture=data|silent|not-installed` (21-27, 81-88) runs the real `after_listing` against a scripted `LegacyTransport` whose replies come from `wire-v1/src/golden_v1.txt` samples `identity_state_seeded` and `ls_state` (31-66; generation 206 gets `ls_state`, other local-state rows get it with `"alice"` replaced by `"old-alias"`, which produces the orphan group). `NoInbox` answers no inbox (68-79). `testdata/identity-backup-v2.json`: alias `fixture-alice`, DSA seed 0..31, KEM seed 100..163; it matches `FIXTURE_TOKEN` (backup_token.rs:249-254, 294).

## 3. Interfaces

**Delegate ops called** (wrappers in ui/src/mail_delegate/ops.rs; `Op` in protocol/src/lib.rs:228-435):

| User action | Call | Where |
|---|---|---|
| Create | `Op::CreateIdentity{alias, description, ui_entropy}` → `Reply::CreatedIdentity{identity, recovery_key}` | login.rs:1551; ops.rs:38-54 |
| Lost create reply | `ListIdentities` twice 3 s apart, then `ExportRecoveryKey` | ops.rs:57, 85-120 |
| Discard a created identity | `Op::DeleteIdentity{addr, expected_gen}` (prompts) | login.rs:1608-1613 |
| Publish | `sign_authority`, `v1_aft_params`, then `NodeAction::CreateContract(Inbox)`, `CreateDelegate`, `CreateContract(AFT)`, `CreateIdentity` | identity_store.rs:1110-1150 |
| Restore | `Op::ImportIdentity{source: Seeds, kind: UserRestore, ..}` → `(info, already_present)` | login.rs:2248-2262 |
| Rename | `Op::UpdateIdentity{addr, alias, description, expected_gen}`; on no answer re-list | identity_store.rs:69-107 |
| Delete | `Op::DeleteIdentity` repeated on `Continue{remaining}` | identity_store.rs:128-158; ops.rs:169-186 |
| Finish deletion, repairs | `finish_deletion`, `Op::RepairIndex{kind, owner}` | identity_store.rs:286-327, 401 |
| Show key | `Op::ExportRecoveryKey{addr}` (consented; 630 s timeout per `ui-state`) | backup_panel.rs:193-197 |
| Revoke "Always allow" | `Op::RevokeExportGrant` | settings.rs:1483 |
| Confirm migrated identity | `ListIdentities`, `Op::ConfirmIdentity`, reload, wait up to 30 s for the AFT record | identity_store.rs:730-773 |
| Select | none: `user.set_logged_id(id)`, `inbox.set_active_id(id)` | login.rs:1191-1194; settings.rs:160-164 |

`ImportSource::RecoveryKey` and `LegacyJson` exist (protocol/src/lib.rs:205-215; delegates/mail-delegate/src/handlers.rs:718-735) but production UI code sends only `Seeds`.

**Contacts.** `NodeAction::CreateContact` → `contact_store::accept` (address book first, then a background save) and a priming GET with `subscribe: true` under the contact's `inbox_wasm_hash` (api.rs:2234-2313), plus a verified-senders push when the bypass is on (2315-2366). `FetchContactKeys` → GET by instance id with `return_contract_code: true` (api.rs:2369-2416); the answer is decoded by `decode_import_fetch` (api.rs:625-677): vk from the contract's `InboxParams.pub_key`, ek and tier policy from the state, `inbox_wasm_hash` = plain `bs58` of `key.code_hash()`. `DeleteContact` → `remove_contact` then delete or rewrite the record (api.rs:2417-2444). Storage: one `ItemKind::Contact` item, `owner: None`, id = `address_of(vk)` (24 bytes); body JSON `ContactRecord{v: 1, local_alias, description, keys: StoredContactKeys, v2?}` (ui/src/app/contact_store.rs:19-29, 59-65, 84-93). `StoredContactKeys` is also the shape the v0.2.x migration decodes (ui/src/migration/legacy_delegates.rs:436-443).

**Settings storage.** `local_state::persist_identity_settings` / `persist_global_settings` patch the snapshot, then under `use-node` do a field-level merged write of the `IdentitySettings` / `GlobalSettings` items (ui/src/local_state.rs:1003-1034; ui/src/local_state/items.rs:666-687). State is keyed by `@<addr hex>` via `state_key(alias)` (local_state.rs:258-267). Types: `freenet_mail_wire_v1::local_state::{IdentitySettings, GlobalSettings, ...}` (wire-v1/src/local_state.rs:256-277).

**Browser / shell.** `postMessage` download and clipboard relays to the gateway origin (login.rs:733-757; ui/src/clipboard.rs:163-210); `FileReader`; `performance.now()`; `window.location.search`.

**Features.** `use-node` (real fetch, login.rs:2496, 2655, 2721, 2737, 2906), not-`use-node` (synthesized contact keys, 2616), `example-data` (`alias_for_encaps_key`, `register_example`, the fixture), `any(use-node, example-data)` for `after_listing` (migration_view.rs:129). Offline builds handle `CreateContact`, `DeleteContact`, `CreateIdentity`, `RenameIdentity`, `DeleteIdentity` in an in-page coroutine and drop everything else, including `UpdateInboxPolicy` (app.rs:215-331).

**DOM contract for Playwright.** Test ids come from `ui/src/testid.rs`, mirrored by hand in `ui/branding/testids.json` and checked by `testid::tests::json_matches_rust_consts` (testid.rs:9-13, 246-249). Several ids are literals: `contact-row`, `contact-fingerprint`, `contact-verify-badge`, `contact-remove` (login.rs:3073-3130), `fm-settings-ident-switcher`, `fm-aft-tier`, `fm-aft-usage-tier` (settings.rs:120, 940, 985). CSS: modals render in both `.fm-pre` and `.fm-app` scopes, and `.bk-*` rules exist in both `ui/public/vendor/css/components/login.css` and `settings.css` (backup_panel.rs:12-14).

## 4. Invariants and where they are enforced

| Rule | Code | Pinned by |
|---|---|---|
| `fmail-key-1:` layout is frozen; a new layout takes a new prefix | protocol/src/recovery_key.rs:21-24 | `fixed_test_vector` (backup_token.rs:317), `future_version_prefix_is_unknown_version` (backup_token.rs:537) |
| v2 JSON backups stay restorable; v1 and v3+ get distinct errors | backup_token.rs:219-241 | `committed_v2_json_fixture_still_restores` (backup_token.rs:575), `backup_from_the_old_exporter_shape_parses` (609), backup_token.rs:593, 629 |
| "Backed up" is recorded only from "I've saved it" or a restore, never from copy or download | backup_panel.rs:6-10; login.rs:517-552 | `backup_stamp_after_create_only_when_backed_up` (login.rs:3173); email-app.spec.ts backup tests |
| The key is in the DOM only while revealed | backup_panel.rs:414-435; login.rs:1663-1668 | `masked_never_contains_the_payload` (backup_panel.rs:633); mail-delegate-identity.spec.ts:39 |
| A stale or late copy never counts against the next key | backup_panel.rs:65-89, 163-178 | backup_panel.rs:518-613 |
| One download per 2.1 s, app-wide | login.rs:636-692 | `download_gap_applies_since_the_last_download` (login.rs:3157) |
| Shell messages go to an explicit origin | login.rs:707-712; clipboard.rs:173-181 | email-app.spec.ts gateway relay tests (per `tests-qa`) |
| One contact per signing key on import; overwrite only when both keys match | address_book.rs:560-613 | address_book.rs:1279, 1292, 1734, 1791 |
| Mail is never sent to a signing key held with two encryption keys | address_book.rs:466-471, 542-552 | address_book.rs:1309, 1750 |
| A recorded recipient is never silently re-pointed | address_book.rs:418-462 | address_book.rs:1439, 1466, 1604, 1616 |
| A relabelled or doubly advertised name does not resolve through `suggested_alias` | address_book.rs:776-792 | address_book.rs:1689 |
| Every contact inbox key is derived with the contact's `inbox_wasm_hash` | address_book.rs:816-817, 906-909 | address_book.rs:1953, 2016; AGENTS.md "Sender-side recipient hash" (correct) |
| The bs58 cache is rebuilt on every address-book write | address_book.rs:473-477, 875-886 | `lookup_by_bs58_cache_invalidates_on_remove_255` (address_book.rs:2050) |
| An unconfirmed identity confirms before sharing or sending | migration_view.rs:55-61; login.rs:1170; settings.rs:1155; app.rs:6168-6169 | migration/tests.rs:2573-2625; guards "the gate asks an unconfirmed identity", "share asks the gate", "send asks the gate" (scripts/check-guards-fail.sh:1205-1212, 1229-1232) |
| Each of login.rs, settings.rs, migration_view.rs has exactly one `ContactCard::from_identity` | login.rs:1022; settings.rs:1140; migration_view.rs:393 | migration/tests.rs:2612-2623 |
| Settings never carries a hard-coded white background | settings.rs | `no_hardcoded_white_backgrounds_340` (settings.rs:1502) |
| Settings edits apply to the identity on screen | settings.rs:37-52, 186-195, 436-450 | email-app.spec.ts settings tests (inventory L115-116) |

## 5. State

- Shipped format: the `fmail-key-1:` codec, the v2 JSON reader, the fingerprint algorithm, the bare-address contact card and `StoredContactKeys` JSON are all in users' hands. The mail delegate these screens call is still a candidate (`published-contract/candidate/`, ui/src/mail_delegate/node.rs:19-22), and `publish` still depends on the transitional `v1_aft_params` (flagged in `ui-state`).
- Exports are recovery keys only (copy or `.txt`). No UI writes the JSON file any more; `ops::export_profile` exists with no UI (ops.rs:555-556).
- Contact sharing here is v1 only: the card is the v1 inbox contract id. v2 addresses and passes are `ui/src/v2/` and Settings → Address.
- Unfinished: see section 10.

## 6. Tests

In-file unit tests (host, no wasm): login.rs 10, address_book.rs 49, backup_token.rs 30, backup_panel.rs 6, settings.rs 1. None is feature-gated, so they run in both configurations:

- `cargo test -p freenet-email-ui --lib app::address_book` (or `app::backup_token`, `app::backup_panel`, `app::login`, `app::settings`); the whole set is in `cargo make test-workspace`.
- `cargo make test-ui-offline` (`--no-default-features --features example-data,no-sync`, Makefile.toml:916-919). Offline, `INBOX_CODE_HASH` is `""` (ui/src/inbox.rs:59-60), which bs58-decodes to an all-zero code key, so the address-derivation tests still pass.

What they prove: the codec and its error taxonomy (backup_token), `CopyTracker` sequences and tracker error classification with a real generational box (backup_panel), the forms' pure helpers (login), and the address book's refusal rules, clash labels, cache and `resolve_for_send` (address_book; every test starts with `clear_for_test`, 1026-1030, because thread-locals survive across tests on one thread). `migration_view` has no in-file tests: `notice_summary` and `gate` are tested in ui/src/migration/tests.rs:755-766 and 2573-2625, the latter by scanning source text. `migration_fixture` is exercised only by `ui/tests/migration-ui.spec.ts`.

Playwright (offline build, `cargo make test-ui-playwright`; needs `cargo make test-ui-playwright-setup` once): `email-app.spec.ts` (create reveal, share modal, import and verify, restore form, backup panel, rename, delete, settings switcher), `mail-delegate-identity.spec.ts`, `export-permission.spec.ts`, `migration-ui.spec.ts`. Real node: `live-node.spec.ts` (`cargo make test-e2e-real-node`). Details are in the `tests-qa` map. Not covered anywhere: Settings → "Restore…", a rejected contact import, contact search, the Advanced screen.

## 7. Changing it safely

- Recovery key: the codec lives in `protocol/`, which the delegate links, so any edit there forces a delegate candidate rebuild (AGENTS.md, `scripts/check-delegate-candidate.sh`). `FIXTURE_TOKEN` is duplicated in ui/tests/email-app.spec.ts:1190.
- `IdentityBackup` / `StoredIdentityKeys` field names are a file format. The delegate has a second parser for the same JSON (handlers.rs:724-735, 16 KiB cap versus the UI's 64 KiB).
- `StoredContactKeys` is read from current items and from v0.2.x data: add fields only with `#[serde(default)]`, never rename.
- `fingerprint_words` output is what users have read to each other, and it names download files and password-manager entries. Do not change the hash, byte order or word list.
- The share address depends on `INBOX_CODE_HASH`. A new inbox generation changes every address a user shares; stored contacts keep working through `inbox_wasm_hash`.
- Source text is pinned: migration_view.rs:56 (`    if confirmed == Some(false) {`) and settings.rs:1155 are guard mutation targets, and migration/tests.rs:2596-2610 searches for `let on_share = move |_|`, `testid::FM_ID_SHARE`, `migration_view::gate(` and `open_share.call(())`. Reformatting or renaming these fails `check-drift`.
- A new or renamed test id must change `ui/branding/testids.json` too. Visible strings are matched by specs (restore errors, notice text, placeholders such as `e.g. Alice (work)`).
- `ALIASES` and `ADDRESS_BOOK` are not reactive: after a write, set `LoginController.updated` and bump `AddressBookGen` (app.rs:233-237; api.rs:2243-2247). Every `ADDRESS_BOOK` mutator must call `bump_address_book_gen`.
- Settings handlers capture a whole `IdentitySettings` at render and write it back; keep the alias read inside the memo and the alias keys on the screens.
- Tasks: a scoped `spawn` dies with its component. Work that must outlive a modal uses `identity_store::spawn_detached` or `spawn_forever` (login.rs:1589, 1607; settings.rs:1481). Late callbacks must `try_write`/`try_peek` (login.rs:2414; backup_panel.rs:325).
- A new modal must be rendered in both `.fm-pre` (login.rs:943-956) and `.fm-app` (app.rs:3476-3481) if it is reachable from both.

## 8. Docs and comments versus code

1. login.rs:64-66 says the AFT tier is hashed into `InboxParams` and frozen. `InboxParams` holds only `pub_key` (wire-v1/src/inbox.rs:32-39) and Settings changes the tier with a `ModifySettings` delta (settings.rs:786-808).
2. login.rs:1002 and app.rs:129-134 say rename is delete + create. It is an in-place `UpdateIdentity` (identity_store.rs:75-80; api.rs:2492).
3. settings.rs:4-10 says settings live in the `mail-local-state` delegate. They are mail-delegate items (items.rs:666-687). Same staleness: login.rs:416 and address_book.rs:1043 ("identity-management delegate"), app.rs:112, 125, 141.
4. UI copy versus where keys live: "Your keys live in this browser" (login.rs:1861), "The keys never leave this browser" on restore (2039, although the seeds are sent to the delegate, 2248-2256), "If you lose this browser's storage, the keys go with it" (settings.rs:513), "no recovery" (login.rs:1889). The reveal card says the true thing (login.rs:1720).
5. settings.rs:530 says the alias "cannot be changed without rotating"; the hub renames it.
6. login.rs:2710 cites `settings.rs:1434` for "verified-only inboxes"; that line is the relay URL handler, and the warning shown to users (2851) describes a recipient-side rule that the sender's tick does not affect.
7. address_book.rs:5 documents `insert_contact(alias, contact)`; it takes a `Contact` (526). `ContactCard::VERSION` is 3 while a test comment speaks of "v1 cards" (1194-1226), and the version never travels.
8. login.rs:376-380: two contradictory doc comments on `remove_in_place`. login.rs:2433-2451: the form's doc comment is attached to the `FetchedKeys` type.
9. Toasts tell the user to "rename it in Contacts" (address_book.rs:511; api.rs:1824); no contact rename exists. docs/qa/manual-test-inventory.md:139 lists "Edit / delete contact" with an edit step.
10. AGENTS.md:836 cites the contact prime GETs at "api.rs ~1442 and ~2540"; they are at api.rs:1833-1836 (`prime_contact_inbox`) and 2271-2284. AGENTS.md's share text "verify: <six words>" is hyphen-joined and always present (login.rs:1028).
11. ui/src/inbox.rs:53-55 says the offline empty code hash fails at `ContractKey::from_params`; it decodes to zero bytes and succeeds.

## 9. Suspected bugs and weak spots

1. **Settings → Account → "Restore…" does nothing visible** (high). It closes Settings and sets `ImportBackup(true)` (settings.rs:465-468), but `ImportForm` renders only inside `IdentifiersList` (login.rs:935-936), which is not mounted while an identity is open (app.rs:374-399). The form appears later, in place of the identity list, when the user logs out.
2. **A rejected contact import is silent** (high). The modal closes and clears itself before the result is known (login.rs:2905-2920); a refusal by `insert_contact` (label taken, signing key already stored) is only `log::error`, which writes to the console (api.rs:2238-2239; ui/src/log.rs:77-93; offline app.rs:232).
3. **Node build: the creation-time backup stamp is filed under the bare alias** (medium). api.rs:2017 calls `record_backup_after_create` before `Identity::set_alias` at 2041, so `state_key` cannot resolve the alias (local_state.rs:258-267) and the snapshot entry is written under it. The delegate write resolves later and is correct, but Settings shows "not backed up" until the next load. The offline coroutine registers first (app.rs:263-265), which is the order the Playwright tests see.
4. **Verifying a clash-labelled contact stores the clash label** (medium). `VerifyContactModal` re-sends the contact as displayed (login.rs:3015-3017) and `contact_store::encode` writes `local_alias` as given (contact_store.rs:67-80), so `bob~a1b2c3` becomes the stored name, with no rename UI to undo it.
5. **Cancel while "Generating…" or while a restore is in flight** (medium-low). Both use a scoped `spawn` (login.rs:1549, 2254) and Cancel is not disabled (1812-1816, 2216-2220). The delegate still creates or imports; the reply, including the one-time recovery key, is dropped, and nothing is published or shown until the next load.
6. **The pasted `verify:` phrase is never compared** with the fetched fingerprint (high that the code does not; login.rs:2575-2585, 2713).
7. **Key cards mislabel the fingerprint** (high). Settings shows words 1-3 under "ML-DSA-65" and 4-6 under "ML-KEM-768" and seeds the KEM grid from the DSA key prefix (settings.rs:383-393, 560-570); the six words are one hash over both keys.
8. Import fetch bookkeeping (low): each keystroke that decodes sends a GET; a superseded address is not cancelled and `fetch_ticks` is not reset (login.rs:2567-2571, 2657-2661).
9. Create and restore do not check for a contact of the same name, unlike rename (login.rs:1065 versus 1537, 2232); the contact is relabelled with a toast.
10. `collect_verified_sender_vk_bytes` is not capped at `MAX_VERIFIED_SENDERS = 1024` (settings.rs:756-762; wire-v1/src/inbox.rs:50), while the delegate allows 2048 contacts.
11. On Cancel the restore textarea's text is dropped without zeroizing (only the success path wipes it, login.rs:2267).

## 10. Open work

- Settings is desktop-only; mobile classes not ported (settings.rs:12-13). "Follow this identity across devices once sync lands" (473).
- No contact rename or edit; no confirmation on contact removal.
- `ContactCard` suggestions never travel, so the branches that read them on import are dead (login.rs:2595-2605); `StoredContactKeys::from_inbox_state` is an unused hook (address_book.rs:1077-1094); the "Migration scaffolding" header is empty (1129).
- JSON profile export has no UI (ops.rs:555). v1 JSON backups cannot be restored (login.rs:485-488).
- The AFT usage panel is not reactive (settings.rs:866-869). `testid.rs` and `testids.json` are synced by hand "until we wire a build-time generator" (testid.rs:11-12).
- A contact's `inbox_wasm_hash` is never refreshed after import (AGENTS.md, correct).
- After a lost create, matching by alias can pick an identity another tab created; a per-create nonce is named as the fix (ops.rs:76-84).

## 11. Questions the tree does not answer

1. Is `owner_ek_bytes` in a fetched inbox state bound to the owner's signature by the v1 contract? `decode_import_fetch` trusts the node's answer.
2. Does the pinned Dioxus version cancel a scoped `spawn` on unmount? Item 9.5 assumes the comment at backup_panel.rs:357-358 is right.
3. What does the delegate answer to an alias over 64 characters on create or rename? The UI checks length only for JSON restores.
4. Are items 9.1 and 9.3 known upstream?
5. `docs/codebase-notes/` exists in the working tree (dated today) although the fork is described as unchanged.

---

## Appendix: the reader's structured summary

The map is complete on disk, so a restart does not need to redo this unit. Nothing was built or run; all findings are from reading.

These are the Dioxus screens outside the mailbox: the first-run and "Your identities" hub (create, restore, rename, delete, back up, share), the contact modals, the eight Settings screens, the recovery-key panel and the v0.2.x migration status view. Two non-UI models also live here: the page's `Identity` list (`ALIASES`) and the address book (six-word fingerprint, contact card, recipient resolution).

Private keys stay in the mail delegate. The page calls CreateIdentity, ImportIdentity (always as raw seeds), UpdateIdentity, DeleteIdentity, ConfirmIdentity, ExportRecoveryKey, RevokeExportGrant and RepairIndex. Exports are the one-line `fmail-key-1:` key only, by copy or `.txt` download. Restore accepts that key or a legacy v2 JSON file, parsed in the page. A contact card is just the bare bs58 v1 inbox contract id; keys, tier policy and `inbox_wasm_hash` come from a GET of that inbox.

Unit coverage of the pure logic is good (96 in-file tests), and Playwright covers most flows offline. I found seven places where UI behaviour looks wrong, the clearest being:
- Settings → "Restore…" does nothing visible while an identity is open.
- A rejected contact import fails silently.
- On a real node, the creation-time "backed up" stamp is filed under the wrong key until reload.

Several comments and much UI copy are stale about where keys live, how rename works and whether the AFT tier is frozen.

### Key facts

- Identity equality and hashing use the ML-DSA vk bytes only, so components must pass the alias as a separate prop to re-render on rename (ui/src/app/login.rs:117-122, 973-980).
- ALIASES is a thread-local Vec, not a signal; upsert replaces by addr and keeps a held entry with a newer generation; re-render needs LoginController.updated (ui/src/app/login.rs:50-54, 160-199, 898-907).
- Identity::from_info hard-codes required_tier Min10 and the default max age regardless of stored settings (ui/src/app/login.rs:88-101).
- Create: Op::CreateIdentity returns the recovery key once; on a lost reply the UI re-lists by alias and uses the consented export; 'Continue to inbox' calls identity_store::publish_or_recover; Discard deletes in the delegate (ui/src/app/login.rs:1551-1562, 1582-1593, 1605-1626; ui/src/mail_delegate/ops.rs:38-54, 85-97).
- Restore parses in the page and always sends ImportSource::Seeds with ImportKind::UserRestore; ImportSource::RecoveryKey and LegacyJson are unused by production UI code (ui/src/app/login.rs:2240-2262; protocol/src/lib.rs:205-215).
- parse_backup is the single restore entry: input starting with '{' is v2 JSON (version must be 2, seeds 32 and 64 bytes, alias at most 64 and description at most 1024 chars), anything else is a recovery key; input over 64 KiB is refused (ui/src/app/backup_token.rs:184-241).
- The recovery key codec is in the protocol crate, shared with the delegate: 'fmail-key-1:' + base58 of [0x01][32][64][4-byte SHA-256 checksum over prefix+body]; pinned by FIXTURE_TOKEN (protocol/src/recovery_key.rs:34-51; ui/src/app/backup_token.rs:294, 317).
- Exports are recovery keys only: ExportedKeyPanel calls Op::ExportRecoveryKey (delegate prompts), the download is '<key>\n' as text/plain named freenet-identity-<alias>.txt, and last_backup_at is stamped only on 'I've saved it' (ui/src/app/backup_panel.rs:185-222, 342-370; ui/src/app/login.rs:521-525, 597-605).
- Downloads are limited to one per 2.1 s app-wide and go to the gateway shell by postMessage to an explicit origin when framed (ui/src/app/login.rs:636-692, 707-757).
- fingerprint_words is BLAKE3(vk || ek), first 12 bytes, six 11-bit BIP-39 indices (66 bits) (ui/src/app/address_book.rs:34-56).
- ContactCard::encode returns only the bs58 inbox ContractInstanceId; version and suggested alias/description never travel; share text is '<address>\nverify: w1-w2-w3-w4-w5-w6' (ui/src/app/address_book.rs:86-111; ui/src/app/login.rs:1021-1036).
- Contact import: FetchContactKeys does a GET with return_contract_code; vk comes from InboxParams.pub_key, ek and tier policy from the state, inbox_wasm_hash from key.code_hash() as plain bs58; the modal polls every 150 ms for 30 s (ui/src/api.rs:625-677, 2369-2416; ui/src/app/login.rs:2494-2561).
- Contacts are stored as one ItemKind::Contact item keyed by address_of(vk), body JSON ContactRecord{v:1, local_alias, description, keys: StoredContactKeys, v2?}; StoredContactKeys is also decoded from v0.2.x data by the migration (ui/src/app/contact_store.rs:19-29, 59-65; ui/src/app/address_book.rs:1050-1070; ui/src/migration/legacy_delegates.rs:436-443).
- insert_contact enforces one contact per signing key and overwrite only when both keys match; load_contact keeps stored duplicates, and resolve_for_send refuses a signing key held with two encryption keys (ui/src/app/address_book.rs:554-618, 466-471, 542-552).
- Drafts record the recipient as 'fmail-addr:' + bs58(address_of(vk)), independent of the inbox contract; resolve_for_send refuses when the typed name now means a different key (ui/src/app/address_book.rs:352-361, 418-462).
- Verification is re-sending NodeAction::CreateContact with verified=true; there is no separate op and no contact rename or edit UI (ui/src/app/login.rs:3014-3019, 3033-3145; ui/src/app/settings.rs:1117-1269).
- Settings writes go through local_state::persist_identity_settings / persist_global_settings (optimistic snapshot, then a field-merged item write under use-node); the AFT screen also sends UpdateInboxPolicy and UpdateVerifiedBypass (ui/src/app/settings.rs:37-60, 790-864; ui/src/local_state.rs:1003-1034).
- A migrated (unconfirmed) identity passes migration_view::gate before share or send; the gate line and the share call sites are pinned by source-text tests and guards (ui/src/app/migration_view.rs:55-61; ui/src/app/settings.rs:1155; ui/src/migration/tests.rs:2573-2625; scripts/check-guards-fail.sh:1205-1212).
- migration_view::after_listing is the whole post-listing sequence: walk, reload, refresh, scrub, confirm_by_inbox, refresh; migration_fixture runs it offline from golden replies via ?migration-fixture=data|silent|not-installed (ui/src/app/migration_view.rs:129-171; ui/src/app/migration_fixture.rs:21-27, 91-97).
- Test ids are constants in ui/src/testid.rs mirrored by hand in ui/branding/testids.json and checked by a unit test; several ids in these screens are string literals (ui/src/testid.rs:9-13, 246-249; ui/src/app/login.rs:3073-3130).

### Docs or comments that disagree with the code

- **[high]** Claim (code comment, ui/src/app/login.rs:64-66): The AFT tier is hashed into the identity's InboxParams and frozen for the inbox lifetime because changing it rotates the contract id. **Reality:** InboxParams holds only pub_key; the tier lives in InboxSettings and Settings changes it with a ModifySettings delta. Evidence: wire-v1/src/inbox.rs:32-39; ui/src/app/settings.rs:786-808
- **[medium]** Claim (code comments, ui/src/app/login.rs:1002 and ui/src/app.rs:129-134): Rename is delete + create with the same key. **Reality:** Rename is an in-place Op::UpdateIdentity with compare-and-swap on the generation. Evidence: ui/src/app/identity_store.rs:75-80; ui/src/api.rs:2492
- **[medium]** Claim (ui/src/app/settings.rs:4-10; ui/src/app/login.rs:416; ui/src/app/address_book.rs:1043; ui/src/app.rs:112, 125, 141): Settings live in the mail-local-state delegate's encrypted secret store; identity keys and contacts are stored in the identity-management delegate. **Reality:** All of these are mail-delegate items (IdentitySettings, GlobalSettings, Contact) written through put_item_merged / contact_store. Evidence: ui/src/local_state/items.rs:666-687; ui/src/app/contact_store.rs:59-65
- **[medium]** Claim (UI copy, ui/src/app/login.rs:1861, 1889, 2039; ui/src/app/settings.rs:513): "Your keys live in this browser"; restore: "The keys never leave this browser"; "If you lose this browser's storage, the keys go with it"; "no recovery". **Reality:** Keys are created and held by the mail delegate on the node; restore sends the seeds to it; a recovery key exists. The reveal card at login.rs:1720 says so correctly. Evidence: ui/src/app/login.rs:1720, 2248-2256
- **[low]** Claim (UI copy, ui/src/app/settings.rs:530): The alias is bound to the identity's keys and cannot be changed without rotating. **Reality:** The hub renames an identity in place. Evidence: ui/src/app/login.rs:1039-1076
- **[low]** Claim (code comment and UI copy, ui/src/app/login.rs:2709-2712, 2851): Verified-only inboxes (settings.rs:1434) will reject sends until the user ticks the verify box. **Reality:** settings.rs:1434 is the relay URL handler. The tick marks the contact verified in the sender's own address book; the only send block it lifts is the sender's own verify_on_send preference. Evidence: ui/src/app/settings.rs:1432-1436; ui/src/app.rs:6263
- **[medium]** Claim (toasts at ui/src/app/address_book.rs:511 and ui/src/api.rs:1824; docs/qa/manual-test-inventory.md:139): Users can rename a clash-labelled contact "in Contacts"; the QA inventory lists an edit-contact recipe. **Reality:** Neither contacts screen has rename or edit; only verify and remove. Evidence: ui/src/app/login.rs:3033-3145; ui/src/app/settings.rs:1223-1262
- **[low]** Claim (ui/src/app/address_book.rs:5, 1194-1226): Public surface: insert_contact(alias, contact); the card is a versioned wire format (v1 cards in a test comment). **Reality:** insert_contact takes a Contact; ContactCard::VERSION is 3 and is never encoded, since encode returns the bare address. Evidence: ui/src/app/address_book.rs:87-94, 526
- **[low]** Claim (AGENTS.md:836): The contact prime GETs are in ui/src/api.rs at ~1442 and ~2540. **Reality:** They are at api.rs:1833-1836 (prime_contact_inbox on load) and 2271-2284 (CreateContact). Evidence: ui/src/api.rs:1836
- **[low]** Claim (code comment, ui/src/inbox.rs:53-55): The offline empty INBOX_CODE_HASH would fail at ContractKey::from_params. **Reality:** bs58-decoding an empty string writes zero bytes and succeeds, giving an all-zero code key; the unguarded address_book tests rely on this in the offline build. Evidence: ui/src/app/address_book.rs:1929
- **[low]** Claim (code comment, ui/src/app/login.rs:376-380): remove_in_place is "Used by NodeAction::DeleteIdentity". **Reality:** It is #[cfg(test)] only; the app removes by address. Evidence: ui/src/app/login.rs:381-400

### Suspected issues (leads from reading, not confirmed)

- **Settings "Restore…" does nothing visible while an identity is open** (confidence: high). The handler closes Settings and sets ImportBackup(true), but ImportForm renders only inside IdentifiersList, which the root mounts only when no identity is open. The form then appears in place of the identity list after a later logout. No Playwright test covers this button. Evidence: ui/src/app/settings.rs:465-468
- **A rejected contact import is silent** (confidence: high). The import modal closes and clears itself before the result is known. A refusal by insert_contact (label taken, signing key already stored, own identity name) is only log::error, which writes to the console; the offline coroutine drops the error too (app.rs:232). Evidence: ui/src/api.rs:2238-2239
- **Node build files the creation-time backup stamp under the bare alias** (confidence: medium). record_backup_after_create runs before Identity::set_alias, so state_key cannot resolve the alias and the snapshot entry is written under the alias string. The delegate write resolves later and is right, but Settings shows 'not backed up' until the next load. The offline coroutine registers first (app.rs:263-265), which is what Playwright sees. Evidence: ui/src/api.rs:2017
- **Verifying a clash-labelled contact stores the clash label as its name** (confidence: medium). VerifyContactModal re-sends the contact as displayed, and contact_store::encode writes local_alias verbatim, so a label like bob~a1b2c3 becomes the stored alias. There is no rename UI to undo it. Evidence: ui/src/app/login.rs:3015-3017
- **Cancel during create or restore orphans the delegate's work** (confidence: low). Both flows use a component-scoped spawn and Cancel stays enabled. If the form unmounts mid-call, the delegate still creates or imports; the reply (including the one-time recovery key) is dropped and nothing is published or shown until the next load. Assumes scoped spawn is cancelled on unmount; not run. Evidence: ui/src/app/login.rs:1549
- **Pasted verify phrase is never compared with the fetched fingerprint** (confidence: high). The import form extracts the 'verify:' line only to decide whether to show a warning; a mismatch with the fingerprint computed from the fetched keys is not detected. Evidence: ui/src/app/login.rs:2575-2585
- **Settings key cards mislabel the fingerprint** (confidence: high). Words 1-3 are shown under 'ML-DSA-65 signing' and 4-6 under 'ML-KEM-768 encryption', and the KEM grid is seeded from the DSA key prefix, but the six words are one hash over both keys. Evidence: ui/src/app/settings.rs:383-393
- **Import fetch bookkeeping leaks and shares a clock** (confidence: low). Each keystroke that decodes sends a new GET; the superseded address is not cancelled in contact_import::PENDING/RESULTS, and fetch_ticks is not reset for the new address. Evidence: ui/src/app/login.rs:2567-2571
- **Verified-senders push is not capped** (confidence: low). collect_verified_sender_vk_bytes sends every verified contact; the contract constant is MAX_VERIFIED_SENDERS = 1024 (wire-v1/src/inbox.rs:50) while the delegate allows 2048 contacts. I did not check what the contract does above the cap. Evidence: ui/src/app/settings.rs:756-762
- **Create and restore skip the contact-name check that rename has** (confidence: low). Rename refuses a contact's name; create and restore check own aliases only, so the same-named contact is relabelled with a toast afterwards. Evidence: ui/src/app/login.rs:1537

### Open work

- Settings is desktop-only; mobile classes not ported (ui/src/app/settings.rs:12-13).
- "These follow this identity across devices once sync lands" (ui/src/app/settings.rs:473).
- No contact rename or edit UI, and no confirmation on contact removal (ui/src/app/login.rs:3127-3137; ui/src/app/settings.rs:1251-1260).
- ContactCard suggestions never travel, so the import branches that read them are dead (ui/src/app/login.rs:2595-2605).
- StoredContactKeys::from_inbox_state is an unused hook (ui/src/app/address_book.rs:1077-1094); the 'Migration scaffolding' header has nothing under it (ui/src/app/address_book.rs:1129).
- JSON profile export has no UI (ui/src/mail_delegate/ops.rs:555-556).
- v1 JSON backups are not restorable (ui/src/app/login.rs:485-488).
- The AFT usage panel is not reactive; it refreshes on the next Settings open (ui/src/app/settings.rs:866-869).
- testid.rs and testids.json are synced by hand 'until we wire a build-time generator' (ui/src/testid.rs:11-12).
- After a lost create, matching by alias can pick an identity another tab created; a per-create nonce is named as the fix (ui/src/mail_delegate/ops.rs:76-84).
- Untested: Settings 'Restore…', a rejected contact import, contact search, the Advanced screen (no matching text in ui/tests/*.spec.ts).

### Open questions

- Is owner_ek_bytes in a fetched v1 inbox state bound to the owner's signature by the contract? decode_import_fetch trusts the node's answer (ui/src/api.rs:625-677).
- Does the pinned Dioxus version cancel a component-scoped spawn on unmount? The cancel-during-create finding depends on it (comment at ui/src/app/backup_panel.rs:357-358 says yes).
- What does the delegate answer to an alias over 64 characters on create or rename? The UI checks length only for JSON restores (ui/src/app/backup_token.rs:203-207).
- Is the Settings 'Restore…' no-op known upstream, and is the node-build backup-stamp ordering intended?
- What does the v1 inbox contract do with more than 1024 verified senders?
- docs/codebase-notes/ exists in the repository working tree (dated today) although the fork is described as having no changes of its own; I did not create or touch it.

### Coverage

Read in full: `ui/src/app/login.rs`, `ui/src/app/address_book.rs (lines 1-1480 and 1919-2103 in full; rest of the test module by name and doc comment)`, `ui/src/app/settings.rs`, `ui/src/app/backup_token.rs`, `ui/src/app/backup_panel.rs`, `ui/src/app/migration_view.rs`, `ui/src/app/migration_fixture.rs`, `ui/src/app/testdata/identity-backup-v2.json`

Skimmed: `ui/src/app/address_book.rs tests at 1480-1918 and 2103-2199 (names and doc comments only)`, `ui/src/app/identity_store.rs (44-163, 265-331, 460-539, 726-812, 1040-1150)`, `ui/src/app/contact_store.rs (1-236, 268-329)`, `ui/src/app.rs (60-416, 3376-3390, 3455-3485, grep hits)`, `ui/src/api.rs (625-756, 970-995, 1795-1836, 1995-2069, 2234-2553, 2713-2787)`, `ui/src/mail_delegate/ops.rs (34-143, 164-188, 544-556)`, `ui/src/local_state.rs (222-267, 677-680, 974-1034)`, `ui/src/local_state/items.rs (100-120, 666-687)`, `ui/src/inbox.rs (53-71, 139-169, 1197-1254)`, `ui/src/clipboard.rs (16-73, 163-210)`, `ui/src/log.rs (74-93)`, `ui/src/migration/mod.rs, ledger.rs, orphans.rs, successor.rs (struct and function signatures)`, `ui/src/migration/tests.rs (750-770, 2568-2640)`, `ui/src/testid.rs (1-30)`, `protocol/src/recovery_key.rs (1-140)`, `protocol/src/lib.rs (200-266, 612-625)`, `wire-v1/src/inbox.rs (24-50)`, `wire-v1/src/local_state.rs (256-277)`, `delegates/mail-delegate/src/handlers.rs (715-735)`, `scripts/check-guards-fail.sh (1200-1240)`, `docs/qa/manual-test-inventory.md (grep hits and 228-252)`, `sibling maps ui-state.md (18-107) and tests-qa.md (grep hits)`

Skipped: (none)
