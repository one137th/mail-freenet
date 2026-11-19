> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# protocol-wire: the UI-delegate protocol crate and the v0.2.x wire types

Status of this map: complete. Every non-test source file under `protocol/` and `wire-v1/` was read in full, plus both `.txt` pins under `ui/src/`. Nothing was built or run. Paths are relative to the repo root.

## 1. Purpose and where this sits

Two crates whose whole job is bytes that other parties parse.

- `protocol/` = `freenet-mail-protocol` (protocol/Cargo.toml:2), a workspace member. The CBOR request/response vocabulary between the page and the mail delegate, plus three shared codecs: the `fmail-key-1:` recovery key, the 24-byte address derivation, and a hand-written client copy of the inbox v2 contract's state types and signed layouts. No freenet-stdlib and no randomness by rule (protocol/Cargo.toml:9-12); deps are blake3, bs58, ciborium, serde, serde_bytes, sha2, zeroize (:13-20).
- `wire-v1/` = `freenet-mail-wire-v1` (wire-v1/Cargo.toml:2), a workspace member. Copies of the v0.2.x contract and delegate types on the workspace's stdlib 0.12.1, because the frozen crates pin stdlib 0.3.5 and the client must link one stdlib (wire-v1/Cargo.toml:9-12, wire-v1/src/lib.rs:1-16).
- `wire-v1/capture/` = `wire-v1-capture`, outside the workspace with its own lockfile (wire-v1/capture/Cargo.toml:9-12). It links the frozen crates by path and prints the golden bytes.
- `ui/src/frozen_keys_v1.txt`, `ui/src/register_defaults_v1.txt`: hex pins read by UI tests.

Consumers: `delegates/mail-delegate` path-depends on BOTH crates (delegates/mail-delegate/Cargo.toml:31-32); `ui` on both, wire-v1 with feature `clock` (ui/Cargo.toml:35-36); `tools/v1-node-seed` on wire-v1 (tools/v1-node-seed/Cargo.toml:14).

## 2. Walkthrough

### protocol/src/lib.rs (727 lines)

Constants (declared here, enforced in the delegate): `PROTOCOL_VERSION: u16 = 1` (:24), `MAX_RESPONSE_BYTES = 512 KiB` (:28), `MAX_ITEM_BYTES = 256 KiB` (:30), `MAX_DECRYPT_BATCH = 32` (:32), `MAX_REMOVAL_BATCH = 1024` (:34), `MAX_RESEALS: u8 = 4` (:438).

Newtypes: `fixed!` gives `RequestId[16]`, `Addr[24]`, `Entropy[32]`, `Hash32[32]`, `PassId[16]` (:36-70); `secret!` gives `DsaSeed[32]`, `KemSeed[64]`, not `Copy`, `Debug` redacted, zeroized on drop (:73-102).

Envelope: `Request { version, request_id, op }` (:105-109); `Response { request_id, result: Result<Reply, Failure> }` (:112-115), with no version field; `Failure { code, detail: String }` (:118-121); `RequestEnvelope { version, request_id, op: ciborium::Value }` with `op()` (:660-671), decoded first so an unknown version or op still yields the id. `to_cbor`/`from_cbor` are plain ciborium (:650-654, :674-676).

`FailureCode`, 10 variants (:123-144): `NotFound`, `Conflict { current_gen }` (0 = absent), `CapReached`, `Refused`, `BadInput`, `Busy`, `Internal`, `UnsupportedVersion`, `UnknownOperation`, `Unreadable` (a write with `expected_gen: None` replaces it).

`ItemKind`, 15 variants (:148-187): `Contact` (no owner; id = 24-byte address); `Draft`, `Sent`, `Kept`, `Archived`, `Flags` (owner = identity; 16-byte id); `IdentitySettings` (owner; empty id); `GlobalSettings` (no owner; empty id); `MigrationJournal` (no owner; empty id; never exported); `MigrationMarker` (no owner; id = predecessor's 32-byte delegate key; never exported); `LegacyOrphan` (no owner; 16 bytes); `InboxV2` (owner; empty id; delegate-written); `PassLedger` (owner; 16-byte pass id; delegate-written); `HeldPass` (owner; id = recipient address; never readable by a page); `Inbound` (owner; 32-byte message id; page may delete). `ItemRef { kind, owner: Option<Addr>, id }` (:189-195), `Item { item, body, generation }` (:197-202).

`Op`, 29 variants (:227-435):

| op | fields | meaning |
|---|---|---|
| GetStatus | | counts, budgets, `v1_transition` |
| ListIdentities | | public info only |
| CreateIdentity | alias, description, ui_entropy | `Conflict` on alias; entropy not all zero; reply carries the recovery key once |
| ImportIdentity | source, kind, alias, description | never overwrites |
| UpdateIdentity | addr, alias, description, expected_gen | CAS |
| DeleteIdentity | addr, expected_gen | prompts every time |
| ConfirmIdentity | addr, expected_gen | a migrated identity is the user's |
| ListItems | kind, owner, after, limit: u16 | id order, paged |
| GetItem | item | |
| PutItem | item, body, expected_gen: Option | `None` create-only; `Some(g)` CAS; `Some(0)` must be absent |
| DeleteItem | item, expected_gen | |
| DecryptV1 | addr, ciphertexts | v1 ciphertexts |
| SealV1Message | addr, recipient_ek, plaintext, ui_entropy | encrypt and sign; nothing signs caller-chosen bytes |
| SignV1Authority | addr, settings_json, last_update_micros: i64 | v1 `sign_payload` |
| SignV1Removal | addr, assignment_hashes | v1 `RemoveMessages` |
| V1AftParams | addr | token-generator parameters (hold the ML-DSA seed) |
| ExportRecoveryKey | addr | prompts |
| ExportProfile | addr | v2 JSON backup; prompts |
| ExportForSuccessor | source_generation: u32, after, session (omitted if None) | paged freenet-migrate export |
| RevokeExportGrant | | drop "Always allow", end session |
| ListDeletingIdentities | | interrupted deletions |
| RepairIndex | kind: Option, owner | reset an unreadable index; prompts |
| RunJobs | | one bounded slice; answers `JobsRun` |
| CreateInboxV2 | addr, ghost_master, existing: Option | first state to PUT; `existing` takes an inbox over |
| UpdateInboxV2 | addr, state, change, ui_entropy | drain, then sign `OwnerChange` |
| IssuePass | addr, contact, invite: Option<u8>, ui_entropy | contact pass or invite |
| HoldPass | addr, pass | keep a pass received out of band |
| SealV2 | from, to, to_state, content, ghost_vk, ui_entropy, reseals (`serde(default)`) | picks admission and `sent_time` |
| RepublishInboxV2 | addr | re-sign from the record; `NotFound` without one |

`Reply`, 22 variants (:574-633): `Status(Status)`, `Identities(Vec<IdentityInfo>)`, `Identity`, `Imported { identity, already_present }`, `Items { items, more }`, `Item(Option<Item>)`, `Ack { generation }`, `Deleted`, `Decrypted(Vec<Result<ByteBuf, FailureCode>>)`, `SealedV1 { content, signature, sender_vk }`, `Signature`, `AftParams`, `RecoveryKey(String)`, `Profile`, `ExportedSecrets { payload, next, session (omitted if None) }`, `Continue { remaining }` (send the same request again), `CreatedIdentity { identity, recovery_key }`, `JobsRun(JobsRun)`, `InboxV2State`, `InboxV2Update(InboxV2Update)`, `PassIssued(Pass)`, `SealedV2(SealedV2)`.

Other types: `ImportSource { RecoveryKey | LegacyJson | Seeds }` (:204-214), `ImportKind { UserRestore | Migration }` (:217-224), `OwnerChange { Drain | AdvanceTime | SetQuarantine | Revoke | BumpEpoch }` (:441-455), `SealContent { Mail | PassUpdate }` (:457-463), `InboxV2Update` (:465-487), `ChangeOutcome { Signed | WaitsForDrain | WaitsForStorage }` (:490-500), `SendAdmission` (:502-507), `SendNotice` (:510-519), `SealedV2` (:521-536), `IdentityInfo` (:538-550), `Status` with `unreadable_identities` omitted when 0 (:552-572), `JobsRun` (:636-647).

### protocol/src/fixed_bytes.rs (20 lines)
`[u8; N]` as a byte string of exactly N bytes; wrong length is `invalid_length` (:14-20).

### protocol/src/address.rs (25 lines)
`ADDRESS_CONTEXT = "freenet-mail address v1"` (:4); `address_of(vk)` = first 24 bytes of `blake3::derive_key` (:7-12). That is all: the text form (36 base58 characters with a 2-byte check, context `freenet-mail address check v1`) is in `ui/src/address.rs:14-40`, and the contract has its own `signed::address_check`.

### protocol/src/recovery_key.rs (250 lines)
`fmail-key-1:` + base58 of `[0x01][dsa 32][kem 64][checksum 4]`; checksum = first 4 bytes of `SHA-256(prefix ‖ body)` (:1-24); `PAYLOAD_LEN = 101` (:44-46); `MAX_TOKEN_CHARS = 512`, checked before base58 because decoding is quadratic and runs per keystroke (:47-51). `encode` (:133-141). `decode` (:150-187) removes all whitespace anywhere, matches the prefix case-insensitively, and checks in order: empty, too long, prefix (`fmail-key-<digits>:` gives `UnknownVersion`), base58, length, checksum, version byte. `TokenError` has 7 variants with user-facing text (:55-119).

### protocol/src/inbox_v2.rs (634 lines)
"Written again from the spec"; does not link the contract (:1-9). `KIND_PASS/OWNER/QUARANTINE = 0/1/2` (:22-24). ML-DSA contexts `fmail/v2/owner`, `fmail/v2/sealed` (:29-31). Numbers (:33-51): `WINDOW_BACK` 2 d, `WINDOW_FORWARD` 400 d, `QUARANTINE_FORWARD` 2 d, `SENT_TIME_GRANULARITY` 3600, `EXPIRY_GRANULARITY` 86400, `CIPHERTEXT_SIZES` 8/16/32 KiB, `PER_PASS` 8, `PER_INVITE` 2, `QUARANTINE_SLOTS` 16, `PER_GHOST` 2, `MAX_REVOKED` 4096, `MAX_TOMBSTONES` 4096, `MAX_INPUT_BYTES` 4 MiB + 256 KiB. `params_bytes` = `0x02 ‖ addr ‖ ghost_master` (:53-62). State types whose field order is the encoding (:64-162): `InboxState`, `InboxDelta`, `OwnerRecord`, `QuarantineMode`, `TimeAttestation`, `Revocation`, `PassUsage`, `PassClass`, `PassCert`, `Message`, `Admission`, `GhostProof`. `encode`/`decode` (:165-182): decode refuses oversize input and any non-canonical encoding. Signed layouts, big-endian, one domain string each (:186-328): `owner_bytes`, `time_bytes`, `cert_bytes`, `revocation_bytes`, `usage_bytes`, `admit_bytes`, `owner_msg_bytes`, `ghost_admit_bytes`, `inner_binding`, `message_id` (blake3). `delivered` = held or tombstoned (:332-335). Envelope types (:337-404): `Pass { cert, pass_sk }` (Debug hides the key), `SealedEnvelope`, `SealedBody`, `EnvelopeKind`, `SystemMsg`, `InboundMessage` (the body of an `ItemKind::Inbound` item). The `len u32 ‖ CBOR ‖ zeros` framing is only described here (:355-356); the code is `delegates/mail-delegate/src/sealing.rs:89-140`.

### protocol/src/golden.rs (591 lines, test only) and golden_v1.txt (95 lines)
Wildcard-free `*_name` matches for `Op`, `Reply`, `FailureCode`, `ItemKind`, `ImportSource`, `ImportKind` (:19-128) and `OwnerChange` (:481-489). `samples()` (:368-479) is a hand-written list; `golden(name)` finds `"<name> <hex>"` anywhere in the file (:508-512). 95 lines = 29 ops + 22 replies + 10 failures + 15 kinds + 3 sources + 2 import kinds + `request` + `response` + 3 optional-field-present forms + 4 changes + 5 extras.

### wire-v1/src (lib.rs 122, golden_samples.rs 558, aft.rs 1259, inbox.rs 370, identity.rs 416, local_state.rs 1127)
- `lib.rs`: module table (:6-11) and the golden test module (:23-122).
- `golden_samples.rs`: `include!`d by both the tests and the capture tool, each supplying modules `inbox`, `aft`, `identity`, `local_state` and a `dsa_signing_key` shim because ml-dsa's seed API differs between rc.8 and 0.1.1 (:3-6). 55 deterministic samples (fixed seeds, timestamp 1_761_000_000, maps of at most one entry, :13-22): 12 AFT, 9 inbox (one is a real ML-DSA signature), 10 identity, 15 local-state, 2 identity states holding the v0.2.x client's `StoredIdentityKeys`/`StoredContactKeys` JSON as text (:408-444), 7 arrays covering every variant of each renamed enum (:446-490). `golden_samples()` attaches a decode/re-encode codec per name; an unlisted name panics (:512-558).
- `aft.rs`: `TokenDelegateMessage` over bincode 1 (:27-51), `Tier` (15 variants, `Ord` by duration, :98-136) with `is_valid_slot`/`normalize_to_next` (:138-342), `TokenDelegateParameters` (:352-393), `DelegateParameters { generator_seed }` (:401-439), `AllocationCriteria` (max age 730 d, :498-513), `TokenAllocationRecord` (:527-677), `TokenAllocationSummary` (:683-737), `TokenAssignment::signature_content` = tier u8 ‖ seconds i64 LE ‖ hash, 41 bytes (:761-783).
- `inbox.rs`: `STATE_UPDATE` salt (:22), `InboxParams` (:36-82), `UpdateInbox` (:86-108), `InboxSettings` (:110-155), `Message` (:168-181), `Inbox` (:183-200), `removal_payload` (no domain tag, :205-207), `sign_payload` = salt ‖ ek ‖ JSON(settings) ‖ micros BE (:223-241), `Inbox::new` behind `clock` (:244-260), `Inbox::with_signature` (:265-279).
- `identity.rs`: `EntryKind`, `AliasInfo`, `IdentityManagement`, `IdentityMsg` (8 variants) (:17-169).
- `local_state.rs`: `Draft`, `SentMessage`, `KeptMessage`, `ArchivedMessage`, settings types, `AliasState`, `LocalState`, `LocalStateMsg` (13 variants) (:28-652).

### wire-v1/capture/src/main.rs (88 lines)
Subcommands `golden`, `kem`, `defaults` (:78-88). Pins stdlib `=0.3.5`, ml-dsa `=0.1.0-rc.8`, ml-kem `=0.3.0-rc.2`, hkdf `=0.12.4`, bincode `=1.3.3` (wire-v1/capture/Cargo.toml:14-28).

### ui/src/frozen_keys_v1.txt, ui/src/register_defaults_v1.txt
Five keys (identity, local-state and token-generator delegates; inbox and AFT-record contracts) derived for fixed inputs (seed `[7;32]`, ui/src/test_util.rs:20-52). Cipher and nonce of stdlib 0.3.5's `RegisterDelegate`, compared with `V0_2_CIPHER`/`V0_2_NONCE` (ui/src/api.rs:603-616).

## 3. Interfaces

- Transport: a `Request` is the payload of an `ApplicationMessage`; the answer is an `ApplicationMessage` marked processed (delegates/mail-delegate/src/handlers.rs:153-165). Page side: `new_request` (ui/src/mail_delegate/mod.rs:118-124), `deliver` (:129-147), timeouts of 30 s, or 630 s for the five prompting ops (:81-90).
- Dispatch: envelope decode (handlers.rs:49-50), strict version equality (:172-183), op decode failure answered `UnknownOperation` (:184-195), then an exhaustive match with no wildcard (:196-304). `v1_transition` gating is the delegate's (handlers.rs:1584-1600); the op is always in the enum.
- CBOR rules (ciborium + serde defaults): enums externally tagged by variant NAME, unit variants a bare text string, others a one-entry map; structs are maps keyed by field name in declaration order; byte fields are byte strings; `Result` is `{"Ok":…}`/`{"Err":…}`. Protocol decoding is by name and ignores unknown fields; `inbox_v2::decode` alone demands byte-exact canonical form.
- wire-v1 encodings: serde_json everywhere except `TokenDelegateMessage` (bincode 1; the workspace stays on bincode 1 for it, Cargo.toml:67-69).
- Who uses wire-v1 today:
  - `aft`, `inbox`: the runtime v1 mail path (ui/src/api.rs, inbox.rs, aft.rs, app/login.rs, app/settings.rs, contact_tier_cache.rs) and the delegate's v1 signing (`sign_payload`, `InboxParams`, `removal_payload`, `STATE_UPDATE`, `DelegateParameters`: handlers.rs:1453-1591).
  - `identity`, and `LocalState`/`LocalStateMsg`: migration only (ui/src/migration/legacy_delegates.rs:23-24, scrub.rs:60-61) plus `tools/v1-node-seed`.
  - `local_state` data types: NOT legacy. They are the page's live model (ui/src/app.rs, ui/src/local_state/items.rs:27-30) and the JSON body of delegate items; the delegate itself writes `KeptMessage` JSON for imports (delegates/mail-delegate/src/watch.rs:442) and the page reads it (ui/src/jobs.rs:78).
- `InboundMessage` CBOR is the body of `Inbound` items, read by the page (ui/src/v2/owner.rs:658).

## 4. Invariants and where each is enforced

1. No rename, reorder-of-fields-that-changes-bytes or retype of a pinned protocol sample: `golden_every_variant_encodes_to_its_pinned_bytes` (golden.rs:514-529); guard "protocol encoding is pinned" (scripts/check-guards-fail.sh:1643-1647).
2. Existing lines of both golden files and both `ui/src` pins never change or disappear: scripts/check-frozen-immutable.sh:123-144, run by `check-drift` (.github/workflows/check-contract-wasm.yml:246-254).
3. A new variant of the seven named enums must be named in golden.rs (compile error under test).
4. A field added later must be `serde(default)` and omitted when empty, so old bytes are unchanged (lib.rs:346-347, :565-566, :609-610).
5. Fixed fields are exact-length byte strings (lib.rs:711-726).
6. The recovery-key layout never changes; a new layout takes a new prefix (recovery_key.rs:22-24); vector at :214-221 and again in ui/src/app/backup_token.rs:294.
7. The address context: guard "inbox v2: the address is the vector's" (check-guards-fail.sh:672-675) via delegates/mail-delegate/src/tests/inbox_vectors.rs:33.
8. inbox v2 field order and layouts equal the contract's vectors: protocol tests (inbox_v2.rs:477-621), guard at check-guards-fail.sh:681-689. `vectors.txt` is never regenerated to pass a test.
9. wire-v1 shapes never change: `copies_reproduce_the_frozen_wire_bytes` (wire-v1/src/lib.rs:80-83) plus `cargo make check-golden-capture` (Makefile.toml:949-960; .github/workflows/build.yml:157).
10. protocol stays free of stdlib and randomness (protocol/Cargo.toml:9-12); the backstop is the delegate import allowlist described in AGENTS.md.

## 5. State

- Frozen in fact: everything in wire-v1 (deployed v0.2.x contracts and delegates parse it), `frozen_keys_v1.txt`, `register_defaults_v1.txt`, and the `fmail-key-1:` format (the comment says users have saved keys, recovery_key.rs:212-213).
- Candidate: the whole `Op`/`Reply` protocol and the inbox v2 types. Only the candidate mail delegate speaks them; `published-contract/wasm/` holds only `v1` and `.release-freeze` blocks releases until v0.3.0. CI already treats `protocol/src/golden_v1.txt` as append-only, so the constraint is self-imposed until the first release.
- Transitional: the five v1 ops, marked for removal with the v1 send path (lib.rs:291, :319-321).

## 6. Tests

- `cargo test -p freenet-mail-protocol` (17 tests). lib.rs: round trip, byte-string encoding, wrong length. address.rs: prefix property. recovery_key.rs: fixed vector, checksum, whitespace and prefix case, version naming. inbox_v2.rs: `params`, two ML-DSA prefixes, 18 layouts, 11 CBOR types against `contracts/inbox-v2/tests/fixtures/vectors.txt` (`include_str!`, :410), one non-canonical refusal. golden.rs: pinned bytes; pinned op and reply bytes decode back to the sample (:531-547); an unknown op still yields its id (:549-583); a flipped byte differs.
- Regenerate: `GOLDEN_PRINT=1 cargo test -p freenet-mail-protocol golden -- --nocapture`, append only new lines (golden.rs:11-13).
- The vectors the protocol tests skip (`address`, `address_check`, `cbor_summary`, `id_quarantine_undecodable`) are covered in delegates/mail-delegate/src/tests/inbox_vectors.rs:33,47,107.
- `cargo test -p freenet-mail-wire-v1` (71 tests): 3 golden tests (both directions; undecodable bytes are an error; a flipped fixture byte fails), 19 AFT, 16 identity, 2 inbox, 31 local-state back-compat and boundary tests.
- `cargo make check-golden-capture`: builds `wire-v1/capture` with `--locked` against the frozen crates and diffs `golden`, `kem`, `defaults` output with the three committed files. Needs bash and the pinned crates.
- `scripts/check-excluded-crates.sh fmt|clippy|test` covers `wire-v1/capture` (:22).
- UI pins: ui/src/inbox.rs:2461-2466 (`use-node` only), ui/src/aft.rs:902, ui/src/local_state.rs:751, ui/src/api.rs:1199 and :1335, ui/src/legacy_registry.rs:106-130.

## 7. Changing it safely

- Adding an `Op` or `Reply`: add the variant; name it in golden.rs; add a sample to `ops()`/`replies()` BY HAND; print and append the line; handle it in the delegate's match (handlers.rs:196-304) and, for a reply, in the page's exhaustive `reply_name` (ui/src/mail_delegate/mod.rs:266); rebuild the delegate candidate. A change under `protocol/` selects the protocol, delegate and ui guard areas (scripts/select-guards.sh:29-32).
- Adding a field to an existing variant: optional and omitted when empty, and add a "present" sample yourself; nothing reminds you.
- Any edit under `protocol/` or `wire-v1/` is an input to the delegate build: `scripts/check-delegate-candidate.sh` must still match, and bytes can change for unobvious reasons (published-contract/candidate/README.md:18-21; scripts/build-wasm.sh:53-59).
- Both crates inherit `version.workspace` and workspace dependency requirements (protocol/Cargo.toml:3,14-20; wire-v1/Cargo.toml:3,15-23), and the delegate's lockfile records them as `0.2.7` (delegates/mail-delegate/Cargo.lock:576-577, :589-590). A workspace version bump or a root requirement change therefore reaches the delegate's `--locked` build.
- Bumping `PROTOCOL_VERSION` is a flag day: the delegate refuses every other version and the page has no code that reacts to `UnsupportedVersion` (no match in `ui/src`).
- wire-v1: never touch a serialized shape. New sample: add it to `golden_samples.rs` with a codec arm, run the capture command, commit the new line; existing lines cannot change, so a changed sample needs a new name. Test order must equal file order (wire-v1/src/lib.rs:55-59).
- A guard that mutates a file under `wire-v1/` needs an `area_of` entry; there is none (scripts/select-guards.sh:37-46).

## 8. Disagreements between docs/comments and code

1. golden.rs:9-11 says a named variant "then fails until it has a sample and a golden line". Not enforced: samples are hand lists and nothing checks that each name has one. `change_name` already has `change_revoke` with no such sample (:486, :465-472).
2. AGENTS.md "pinned per variant": true for `Op`, `Reply`, `FailureCode`, `ItemKind`, `ImportSource`, `ImportKind`. `ChangeOutcome::Signed` and `WaitsForDrain` have no pinned bytes (only `WaitsForStorage`, golden.rs:352), and `SealedEnvelope`, `SealedBody`, `EnvelopeKind`, `SystemMsg`, `InboundMessage` are pinned nowhere.
3. "Every request gets exactly one response" (lib.rs:3-6, AGENTS.md): a payload whose envelope does not decode returns `DelegateError::Deser` and no `Response` (handlers.rs:49-50). A known op with malformed fields is answered `UnknownOperation`, not `BadInput` (handlers.rs:184-194).
4. inbox_v2.rs:171-172 says `decode` refuses "as the contract does". The contract also refuses over-count maps while decoding (contracts/inbox-v2/src/types.rs:27-31, :104-116); this copy has no `MAX_MESSAGES` (88) or `MAX_USAGE` (1024) and no bounded maps.
5. wire-v1/capture/src/main.rs:14 "CI runs both": there are three commands and CI runs all three (Makefile.toml:956-958); the task's description and echo say only "golden and KEM" (:950, :959).
6. delegates/mail-delegate/Cargo.toml:9-11 says its graph "must not move with workspace dependency bumps" and that every dependency is pinned exactly; `protocol` and `wire-v1` bring workspace-inherited requirements and version into it (section 7).
7. "Copied verbatim" (wire-v1/src/inbox.rs:2): the copy adds `removal_payload`, `with_signature`, a `pub` `STATE_UPDATE` and the `clock` gate. `aft.rs` differs from the frozen source only in the ml-dsa seed call and removed wasm logging.
8. Test name `golden_bytes_decode_and_re_encode_unchanged` only decodes and compares, and only ops and replies (golden.rs:539-546).

## 9. Suspected bugs and weak spots

- Unpinned cross-version formats (high confidence they are unpinned): the sealed envelope travels between different users' delegates and `InboundMessage` is stored and read by the page, yet a field rename in either passes every test in these crates.
- A newer delegate's `Reply` or `FailureCode` variant makes the page's `from_cbor::<Response>` fail; it logs "undecodable" and the call ends as a 30 s `Unknown` (ui/src/mail_delegate/mod.rs:138-141). The tolerant envelope exists only in the request direction. High confidence on behaviour.
- Workspace version bump versus the delegate lockfile and candidate bytes (medium; not run). RELEASING.md:117-122 refreshes only the root lockfile.
- `Tier::Min30` normalizes with base 15 (wire-v1/src/aft.rs:233, same in the frozen source), while `is_valid_slot` uses 30 (:148). The client never calls `normalize_to_next`; the frozen token generator does (modules/antiflood-tokens/delegates/token-generator/src/lib.rs:366), so Min30 slots can land on :15/:45 and be rejected. Medium confidence; not fixable without a new generation.
- The rule numbers in inbox_v2.rs:33-51 are hand copies with no equality test against `merge::Rules::V1`; `PER_GHOST` and `EXPIRY_GRANULARITY` have no users.
- `recovery_key::encode`/`decode` leave seeds in unzeroized `Vec`/`String` temporaries (:134-140, :151, :169); `Reply` and `ImportSource` derive `Debug` over recovery-key strings (lib.rs:204, :574). Low; the page avoids `{:?}` on replies.
- `Inbox::new` and the `clock` feature have no caller in the tree. Low.

## 10. Open work

- Remove the v1 ops and `Status.v1_transition` with the v1 send path (lib.rs:291, :319-321).
- No TODO/FIXME markers in either crate. Documented limits copied from v0.2.x: delivery-state gaps (#58, local_state.rs:25-27), unarchive (#60, :598-601).

## 11. Questions the tree does not answer

- How `ui/src/frozen_keys_v1.txt` was produced: the capture tool has no subcommand for it, so it cannot be re-derived from the frozen crates in CI.
- Whether `fmail-key-1:` shipped in a tagged v0.2.x release (no tags in this clone).
- Whether the envelope and `InboundMessage` are meant to be frozen at v0.3.0 without golden bytes.
- Whether the release procedure intends a delegate lockfile refresh and candidate rebuild after the version bump and before freezing `wasm/v2/`.

---

## Appendix: the reader's structured summary

Two crates that define bytes other parties parse. `protocol/` (freenet-mail-protocol) is the CBOR request/response vocabulary between the page and the mail delegate: 29 `Op` variants, 22 `Reply` variants, 10 `FailureCode`s, 15 `ItemKind`s, a version-1 envelope, plus the `fmail-key-1:` recovery-key codec, the 24-byte address derivation and a hand-written client copy of the inbox v2 contract's state types and signed layouts. `wire-v1/` (freenet-mail-wire-v1) holds copies of the v0.2.x inbox, AFT, identity-delegate and local-state types on the workspace stdlib; `wire-v1/capture` links the frozen crates to produce the golden bytes those copies are tested against.

State: wire-v1 is frozen in fact (deployed v0.2.x code parses it). The protocol and inbox v2 types are a candidate: only the unreleased mail delegate speaks them and `.release-freeze` blocks releases, but CI already treats `protocol/src/golden_v1.txt` as append-only. Both crates are path dependencies of the mail delegate, so edits here reach the delegate candidate build.

Main weak spots: golden coverage is by hand-listed samples, so two `ChangeOutcome` variants and the sealed-envelope and `InboundMessage` types have no pinned bytes; version tolerance exists only in the request direction; a workspace version bump likely invalidates the delegate lockfile and candidate. Nothing was built or run.

On restarting: the map file is complete on disk and self-contained, so a rerun of this unit can reuse it rather than re-read the code.

### Key facts

- Protocol constants are declared in the crate and enforced in the delegate: PROTOCOL_VERSION=1, MAX_RESPONSE_BYTES=512 KiB, MAX_ITEM_BYTES=256 KiB, MAX_DECRYPT_BATCH=32, MAX_REMOVAL_BATCH=1024, MAX_RESEALS=4 (protocol/src/lib.rs:24-34, :438).
- Envelope: Request{version,request_id,op}, Response{request_id,result} with no version field, Failure{code,detail}; RequestEnvelope keeps `op` as ciborium::Value so an unknown version or op still yields the request id (protocol/src/lib.rs:105-121, :660-671).
- The delegate requires version equality (UnsupportedVersion otherwise), answers any op decode failure with UnknownOperation, then matches Op exhaustively with no wildcard (delegates/mail-delegate/src/handlers.rs:172-195, :196-304).
- Op has 29 variants, Reply 22, FailureCode 10, ItemKind 15 (protocol/src/lib.rs:227-435, :574-633, :123-144, :148-187); the full one-line-each tables are in the map.
- CBOR rules: ciborium with serde defaults, enums externally tagged by variant name, structs as maps in declaration order, byte fields as exact-length byte strings via fixed_bytes (protocol/src/fixed_bytes.rs:10-20; protocol/src/lib.rs:36-42, :711-726). Later-added fields are serde(default) and omitted when empty (lib.rs:346-347, :565-566, :609-610).
- golden.rs has wildcard-free name matches for Op, Reply, FailureCode, ItemKind, ImportSource, ImportKind and OwnerChange, but the samples are hand-written lists; golden_v1.txt has 95 lines looked up by name anywhere in the file (protocol/src/golden.rs:19-128, :368-479, :481-489, :508-512).
- Both golden files and both ui/src pins are line-append-only in CI: scripts/check-frozen-immutable.sh:123-144, run by check-drift (.github/workflows/check-contract-wasm.yml:246-254). Guard 'protocol encoding is pinned' mutates a FailureCode rename (scripts/check-guards-fail.sh:1643-1647).
- Recovery key: `fmail-key-1:` + base58 of [0x01][dsa 32][kem 64][4-byte SHA-256 checksum over prefix+body], 101 payload bytes, input capped at 512 chars before base58; decode strips all whitespace and matches the prefix case-insensitively (protocol/src/recovery_key.rs:1-24, :44-51, :150-187). Fixed vector at :214, duplicated in ui/src/app/backup_token.rs:294.
- address.rs is derivation only: first 24 bytes of blake3::derive_key("freenet-mail address v1", vk) (protocol/src/address.rs:4-12). The text form with its 2-byte check lives in ui/src/address.rs:14-40.
- inbox_v2.rs is a hand copy of the contract's types and ten signed layouts, checked against contracts/inbox-v2/tests/fixtures/vectors.txt (params, 2 ML-DSA prefixes, 18 layouts, 11 CBOR types); decode requires canonical re-encoding and at most 4 MiB + 256 KiB (protocol/src/inbox_v2.rs:53-62, :165-182, :186-328, :410, :477-621). The four vectors it skips are covered in delegates/mail-delegate/src/tests/inbox_vectors.rs:33,47,107.
- The mail delegate path-depends on both crates, and they inherit the workspace version and dependency requirements; its lockfile records them as 0.2.7 (delegates/mail-delegate/Cargo.toml:31-32; protocol/Cargo.toml:3,14-20; wire-v1/Cargo.toml:3,15-23; delegates/mail-delegate/Cargo.lock:576-577, :589-590). Any edit under protocol/ or wire-v1/ is an input to the candidate build (published-contract/candidate/README.md:18-21).
- wire-v1 copies four frozen crates: aft (modules/antiflood-tokens/interfaces), inbox (contracts/inbox), identity (modules/identity-management), local_state (modules/mail-local-state) (wire-v1/src/lib.rs:6-11). Encodings are serde_json except TokenDelegateMessage, which is bincode 1 (wire-v1/src/aft.rs:39-51; Cargo.toml:67-69).
- wire-v1 golden: golden_samples.rs is include!d by both the crate's tests and wire-v1/capture; 55 deterministic samples; the test demands the same names in the same order, identical bytes, and decode/re-encode identity (wire-v1/src/golden_samples.rs:3-22, :512-558; wire-v1/src/lib.rs:49-83).
- Capture tool: subcommands golden, kem, defaults against stdlib =0.3.5, ml-dsa =0.1.0-rc.8, ml-kem =0.3.0-rc.2 (wire-v1/capture/src/main.rs:33-88; wire-v1/capture/Cargo.toml:14-28). `cargo make check-golden-capture` diffs all three outputs with the committed files (Makefile.toml:949-960), run in .github/workflows/build.yml:157.
- Who uses wire-v1: aft and inbox are the runtime v1 mail path in the UI and the delegate's v1 signing (delegates/mail-delegate/src/handlers.rs:1453-1591); identity and LocalState/LocalStateMsg are migration-only (ui/src/migration/legacy_delegates.rs:23-24, scrub.rs:60-61) plus tools/v1-node-seed.
- wire-v1 local_state data types are not legacy: they are the page's live model and the JSON body of delegate items; the delegate writes KeptMessage JSON for imports (delegates/mail-delegate/src/watch.rs:442; ui/src/local_state/items.rs:27-30; ui/src/jobs.rs:78).
- ui/src/frozen_keys_v1.txt pins five keys derived with seed [7;32] and is read by tests at ui/src/inbox.rs:2461-2466, ui/src/aft.rs:902, ui/src/local_state.rs:751, ui/src/api.rs:1199 and :1335, ui/src/legacy_registry.rs:106-130 (helper ui/src/test_util.rs:20-52). register_defaults_v1.txt pins stdlib 0.3.5's RegisterDelegate cipher and nonce (ui/src/api.rs:603-616).
- Page side of the protocol: new_request stamps PROTOCOL_VERSION and a random id; deliver reads the id first, then decodes the whole Response; timeouts are 30 s, or 630 s for the five prompting ops; reply_name is another exhaustive match over Reply (ui/src/mail_delegate/mod.rs:81-90, :118-147, :266).
- Nothing speaking this protocol has shipped: published-contract/wasm/ holds only v1, the delegate is under published-contract/candidate/, and .release-freeze blocks releases until v0.3.0.
- Test counts: 17 in freenet-mail-protocol, 71 in freenet-mail-wire-v1 (3 golden, 19 AFT, 16 identity, 2 inbox, 31 local-state). A guard mutating a wire-v1 file would need a new area_of entry; none exists (scripts/select-guards.sh:37-46).

### Docs or comments that disagree with the code

- **[medium]** Claim (code comment, protocol/src/golden.rs:9-11): A new variant does not compile until it is named, "and the test then fails until it has a sample and a golden line". **Reality:** Only the naming is enforced. Samples are hand-written lists and nothing checks that every name has a sample or a golden line; `change_name` already names `change_revoke`, which has no sample of its own. Evidence: protocol/src/golden.rs:368-479, :465-472, :486
- **[medium]** Claim (AGENTS.md, section "The mail delegate", first paragraph): The protocol's encoding is pinned per variant in protocol/src/golden_v1.txt. **Reality:** True for Op, Reply, FailureCode, ItemKind, ImportSource and ImportKind. ChangeOutcome::Signed and WaitsForDrain have no pinned bytes (only WaitsForStorage appears, inside reply_inbox_v2_update), and SealedEnvelope, SealedBody, EnvelopeKind, SystemMsg and InboundMessage are pinned neither here nor in the contract vectors. Evidence: protocol/src/golden.rs:352; protocol/src/lib.rs:490-500; protocol/src/inbox_v2.rs:357-404
- **[low]** Claim (protocol/src/lib.rs:3-6 and AGENTS.md "Responses" bullet): Every request gets exactly one response. **Reality:** A payload whose envelope does not decode returns DelegateError::Deser and no Response. A known operation with malformed fields is answered UnknownOperation rather than BadInput. Evidence: delegates/mail-delegate/src/handlers.rs:49-50, :184-194
- **[low]** Claim (code comment, protocol/src/inbox_v2.rs:171-172): `decode` refuses over-limit and non-canonical input "as the contract does". **Reality:** The contract also refuses maps over their count bounds while decoding (88 messages, 1024 usage entries, 4096 revocations and tombstones). The protocol copy has no MAX_MESSAGES or MAX_USAGE and no bounded maps. Evidence: contracts/inbox-v2/src/types.rs:27-31, :104-116; protocol/src/inbox_v2.rs:173-182
- **[low]** Claim (code comment, wire-v1/capture/src/main.rs:14): "CI runs both and fails if the output differs" (of the capture commands). **Reality:** The file lists three commands (golden, kem, defaults) and CI runs all three. The cargo-make task's description and final echo mention only the golden and KEM fixtures. Evidence: Makefile.toml:950, :956-959
- **[medium]** Claim (code comment, delegates/mail-delegate/Cargo.toml:9-11): The delegate's build graph must not move with workspace dependency bumps; every dependency that reaches the WASM is pinned exactly. **Reality:** Its two path dependencies, protocol and wire-v1, take their version and dependency requirements from the root workspace, and the delegate's lockfile records them at the workspace version 0.2.7. Only the lockfile pins them. Evidence: protocol/Cargo.toml:3,14-20; wire-v1/Cargo.toml:3,15-23; delegates/mail-delegate/Cargo.lock:576-577, :589-590
- **[low]** Claim (module docs, wire-v1/src/inbox.rs:1-4 and wire-v1/src/lib.rs:3-4): The wire-v1 modules are "copied verbatim minus the contract implementation". **Reality:** inbox.rs adds removal_payload and Inbox::with_signature, makes STATE_UPDATE public and gates Inbox::new behind the `clock` feature. aft.rs differs from the frozen source only in the ml-dsa seed call and removed wasm logging (checked by diff). Evidence: wire-v1/src/inbox.rs:22, :205-207, :244, :265-279; wire-v1/src/aft.rs:413-417
- **[low]** Claim (test name, protocol/src/golden.rs:532): Test `golden_bytes_decode_and_re_encode_unchanged`. **Reality:** It only decodes the pinned bytes and compares with the sample, and only for ops and replies; failures, kinds and the extra samples are never decoded. Evidence: protocol/src/golden.rs:539-546

### Suspected issues (leads from reading, not confirmed)

- **Cross-version formats with no pinned bytes** (confidence: high). The sealed envelope (SealedEnvelope, SealedBody, EnvelopeKind, SystemMsg) travels between different users' delegates, and InboundMessage is stored by the delegate and read by the page. Neither has a golden line or a contract vector, so a field rename or reorder passes every test in these crates. ChangeOutcome::Signed and WaitsForDrain are also unpinned. Evidence: protocol/src/inbox_v2.rs:357-404; protocol/src/golden.rs:352; ui/src/v2/owner.rs:658
- **A forgotten golden sample is not detected** (confidence: high). Adding a variant and naming it in golden.rs satisfies the compiler; if no sample is added to the hand-written lists, the test still passes and the variant ships unpinned. There is also no check that every golden line still has a sample. Evidence: protocol/src/golden.rs:174-288, :290-366, :514-529
- **Version tolerance exists only for requests** (confidence: high). The page decodes the whole Response strictly. A Reply or FailureCode variant from a newer delegate fails to decode, is logged as undecodable and the call ends as a 30 s Unknown. Nothing in ui/src reacts to UnsupportedVersion or UnknownOperation, and the delegate accepts only version equality, so a PROTOCOL_VERSION bump is a flag day. Evidence: ui/src/mail_delegate/mod.rs:129-147; delegates/mail-delegate/src/handlers.rs:172-183
- **Workspace version bump likely breaks the delegate's locked build and changes the candidate** (confidence: medium). protocol and wire-v1 use version.workspace, and the delegate's lockfile records them as 0.2.7. Bumping the workspace version for a release should make the delegate's --locked build, tests and clippy fail until its lockfile is refreshed, and should change the candidate bytes. RELEASING.md's bump step refreshes only the root lockfile. Not run; inferred from the manifests. Evidence: delegates/mail-delegate/Cargo.lock:576-577, :589-590; RELEASING.md:117-122; scripts/check-excluded-crates.sh:38-40
- **Tier::Min30 normalizes to a 15-minute base** (confidence: medium). normalize_to_next uses base 15 for Min30 while is_valid_slot requires a multiple of 30, so a Min30 slot can land on :15 or :45 and fail validation. The copy matches the frozen source. The client never calls normalize_to_next; the frozen token-generator delegate does, so the defect is in shipped v1 WASM and cannot be fixed without a new generation. Evidence: wire-v1/src/aft.rs:233 and :148; modules/antiflood-tokens/delegates/token-generator/src/lib.rs:366
- **inbox v2 rule numbers are hand copies with no equality test** (confidence: medium). The window and cap constants duplicate the contract's by hand; only byte vectors and the delegate's behavioural interop tests connect them. PER_GHOST and EXPIRY_GRANULARITY have no users outside their definitions. Evidence: protocol/src/inbox_v2.rs:33-51; contracts/inbox-v2/src/merge.rs:104-114
- **Seed material left in unzeroized temporaries and Debug-printable types** (confidence: low). recovery_key encode and decode hold the seeds in ordinary Vec and String buffers that are not zeroized, although DsaSeed and KemSeed are. Reply and ImportSource derive Debug over recovery-key strings; the page avoids Debug-formatting replies today. Evidence: protocol/src/recovery_key.rs:134-140, :151, :169; protocol/src/lib.rs:204, :574
- **Dead code: Inbox::new and the `clock` feature** (confidence: medium). No caller of Inbox::new exists in ui, tools or the delegate; every user builds states with Inbox::with_signature. The UI still enables the feature. Evidence: wire-v1/src/inbox.rs:244-260; ui/Cargo.toml:35

### Open work

- The five v1 operations (DecryptV1, SealV1Message, SignV1Authority, SignV1Removal, V1AftParams) and Status.v1_transition are transitional, to be removed with the v1 send path (protocol/src/lib.rs:291, :319-321, :561).
- The protocol and inbox v2 types become frozen only when v0.3.0 ships the delegate; until then .release-freeze blocks releases and the candidate must be rebuilt after edits here (published-contract/candidate/README.md:18-21).
- No TODO or FIXME markers exist in protocol/ or wire-v1/.
- Limits documented in the copied v0.2.x types: sent rows can stay Pending (#58, wire-v1/src/local_state.rs:25-27); true unarchive is not implemented (#60, wire-v1/src/local_state.rs:598-601, :607-608).
- No guard in check-guards-fail.sh mutates a wire-v1 file, and select-guards.sh has no area_of entry for wire-v1 (scripts/select-guards.sh:37-46).

### Open questions

- How was ui/src/frozen_keys_v1.txt produced? wire-v1/capture has no subcommand for it, so CI cannot re-derive it from the frozen crates the way it does the other fixtures.
- Did the `fmail-key-1:` format ship in a tagged v0.2.x release? The comment at protocol/src/recovery_key.rs:212-213 says users have saved keys, but this clone has no tags to confirm it.
- Are the sealed envelope and InboundMessage meant to be frozen at v0.3.0 without golden bytes, or is pinning them planned before the release?
- Does the release procedure intend a delegate lockfile refresh and candidate rebuild after the workspace version bump and before the candidate is frozen into published-contract/wasm/v2/? I could not run cargo to confirm the --locked failure.
- Is strict version equality in the delegate the intended long-term policy, given that the UI and delegate are said to be released separately (protocol/src/golden.rs:3-5) and the page has no handling for UnsupportedVersion?

### Coverage

Read in full: `protocol/Cargo.toml`, `protocol/src/lib.rs`, `protocol/src/fixed_bytes.rs`, `protocol/src/address.rs`, `protocol/src/recovery_key.rs`, `protocol/src/inbox_v2.rs`, `protocol/src/golden.rs`, `wire-v1/Cargo.toml`, `wire-v1/src/lib.rs`, `wire-v1/src/golden_samples.rs`, `wire-v1/src/aft.rs`, `wire-v1/src/inbox.rs`, `wire-v1/src/identity.rs`, `wire-v1/src/local_state.rs`, `wire-v1/capture/Cargo.toml`, `wire-v1/capture/src/main.rs`, `ui/src/frozen_keys_v1.txt`, `ui/src/register_defaults_v1.txt`, `scripts/check-frozen-immutable.sh`, `delegates/mail-delegate/Cargo.toml`

Skimmed: `protocol/src/golden_v1.txt (all 95 lines; hex of the 18 longest lines truncated at 230 characters)`, `wire-v1/src/golden_v1.txt (all 55 names, byte lengths and the first 40 bytes of each line)`, `delegates/mail-delegate/src/handlers.rs (lines 30-204 and 1436-1607)`, `delegates/mail-delegate/src/watch.rs (lines 299-310 and 425-452)`, `delegates/mail-delegate/src/tests/inbox_vectors.rs (lines 1-60 and test names)`, `delegates/mail-delegate/Cargo.lock (protocol and wire-v1 entries only)`, `ui/src/mail_delegate/mod.rs (lines 1-215 and 262-276)`, `ui/src/address.rs (lines 1-60)`, `ui/src/test_util.rs (lines 1-52)`, `ui/src/api.rs (lines 560-640 and 1180-1200)`, `ui/src/local_state/items.rs (lines 1-33 and 120-130)`, `ui/src/inbox.rs (lines 34-42 and 2448-2470)`, `ui/src/legacy_registry.rs (lines 100-132)`, `contracts/inbox-v2/src/types.rs (lines 20-66 and 252-275)`, `contracts/inbox-v2/src/merge.rs (lines 55-116)`, `contracts/inbox-v2/tests/fixtures/vectors.txt (vector names only)`, `modules/antiflood-tokens/delegates/token-generator/src/lib.rs (lines 350-400)`, `frozen sources modules/antiflood-tokens/interfaces/src/lib.rs, contracts/inbox/src/lib.rs, modules/identity-management/src/lib.rs, modules/mail-local-state/src/lib.rs (by diff against the wire-v1 copies)`, `Cargo.toml (lines 1-110)`, `Makefile.toml (lines 907-965)`, `scripts/select-guards.sh (lines 20-48)`, `scripts/check-guards-fail.sh (lines 665-700, 1608-1655)`, `scripts/check-excluded-crates.sh (lines 1-60)`, `scripts/build-wasm.sh (lines 45-70)`, `.github/workflows/build.yml (lines 135-165)`, `.github/workflows/check-contract-wasm.yml (lines 232-258)`, `published-contract/candidate/README.md (lines 1-40)`, `RELEASING.md (lines 105-123)`, `.release-freeze`

Skipped: `wire-v1/capture/Cargo.lock (lockfile; excluded by the reading rule, checked only for the freenet-stdlib 0.3.5 entry)`, `wire-v1/capture/target/ (build output; instructed not to touch target directories)`
