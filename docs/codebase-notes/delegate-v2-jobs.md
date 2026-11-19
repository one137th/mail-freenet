> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Map: mail delegate, inbox v2 operations, sealing, background jobs (key: delegate-v2-jobs)

Repo root `/home/user/mail-freenet`. Abbreviations for `path:line` references:
`J` = `delegates/mail-delegate/src/jobs.rs`, `W` = `.../src/watch.rs`, `S` = `.../src/sealing.rs`,
`V` = `.../src/inbox_v2.rs`, `IJ` = `.../src/inbox_jobs.rs`, `H` = `.../src/handlers.rs`,
`ST` = `.../src/store.rs`, `T/` = `.../src/tests/`, `P` = `protocol/src/lib.rs`,
`PW` = `protocol/src/inbox_v2.rs`, `G` = `scripts/check-guards-fail.sh`.

## 1. Purpose and place

Five source files of the out-of-workspace crate `delegates/mail-delegate` (own `Cargo.lock`, exact pins, `delegates/mail-delegate/Cargo.toml:9-16`). They are (a) everything the delegate does with no page open and (b) every inbox v2 step that needs a secret. `handlers::process` is the only entry: page requests are origin-gated and dispatched to `inbox_v2::*` (H:240-281) or `run_jobs` (H:299, H:324-332); the host's `NodeStarted`, `WakeupFired`, `ContractNotification`, `GetContractResponse` go to `background()` then `watch::run` (H:62-81, H:97-101); `SubscribeContractResponse` and `UpdateContractResponse` are dropped unread (H:85-88). Manifest: `lifecycle=[NodeStarted]`, `capabilities=[Background]`, wake-ups `rescan=900`, `v1watch=3600`, `renew=21600`, `clock=86400` seconds (`delegates/mail-delegate/src/lib.rs:42-46`). Contract rules, wire types and the UI callers (`ui/src/v2/`, `ui/src/jobs.rs`) are in sibling maps.

## 2. Walkthrough

### jobs.rs (468 lines): pure planner
- Numbers: `MAX_SUBSCRIPTIONS 256` (J:21), `MAX_OPS_PER_MINUTE 300` (J:23), `MAX_SCANS_PER_RUN 4` (J:25), duty `10 ms/s` refill, `10_000 ms` burst (J:27-28), `SCAN_COST_MS 10` (J:37), `MESSAGE_COST_MS 120` (J:43), `MAX_MESSAGES_PER_RUN 12` (J:49), `SCAN_MS_PER_MIB 150` (J:54), `CLOCK_RESET_SECS 86_400` (J:61), missing-inbox backoff `3600` doubling to `86_400` (J:64-65), `MAX_MISSING 256` (J:67), `MAX_WRITES_PER_MINUTE 60` (J:70).
- `Tick::ALL` tags `rescan v1watch renew clock` must equal the manifest's (J:82-87). `Trigger { NodeStarted, Tick, Contract(id), RunJobs(origin) }`; only `RunJobs` is not "unprompted" (J:99-113).
- `jobs_for` (J:137-164): NodeStarted = V1Refresh, V1Scan, DrainOwn, AdvanceClock, Renew; RunJobs = those plus `SendQueued`; Tick(V1Watch) = V1Refresh, V1Scan; Tick(Rescan) = V1Scan, DrainOwn, AdvanceClock, Renew; Contract = V1Scan, DrainOwn, Renew; Tick(Renew) = Renew; Tick(Clock) = DrainOwn, AdvanceClock.
- `rank_subscriptions` (J:201-233): tiers `Own < V1Inbox < Contact`, origins take turns inside a tier, ids deduplicated, cut at 256.
- `Budget` (J:238-257; CBOR under `bg/budget`): `minute, ops, duty_ms, duty_at, cursor, missing[(id, asks, next)], writes[(id, n)], cursor_v2`; only `writes` and `cursor_v2` are `#[serde(default)]`. `advance` (J:278) resets everything but the cursors when the clock is more than a day behind `duty_at`. `message_slice` = 12 for a page, `min(12, duty_ms/120)` for a node run (J:365).
- `plan` (J:401-468): eligible targets by trigger; scans rotate from `cursor`, at most 4, each charged 10 ms only when unprompted; the cursor does not move for `Contract` (J:434). Subscriptions only with V1Refresh: v1 targets plus v2 candidates, ranked, and for a node run filtered by `may_ask` and cut to the ops left (J:439-460). `fetch_missing = refresh` (J:465).

### watch.rs (452 lines): `run`, targets, budget storage, v1 scan
- Names: `GLOBAL = Origin([0;32])` (W:48), `ix/bg/origins`, `bg/budget` there (W:50-51), `bg/cursor` in the page's origin (W:53), `MAX_ORIGINS 16` (W:56), `SEEN_PER_SHARD 512` (W:58), `MAX_INBOX_BYTES 16 MiB` (W:61).
- `run` (W:89-199): `now = now_secs().unwrap_or(0)`; `targets()` lists origins (the page's own, or `ix/bg/origins`), every identity's v1 `Target` and v2 `Own` (W:204-253), dropping origins whose identity index is empty; budget loaded and advanced; a page run swaps in its own cursor and restores the shared one (W:99-118); **budget saved before any import** (W:119-124); v1 scans; `inbox_jobs::run` (W:143); `note_held` for everything read (page runs too, W:146-148); GETs only when `unprompted && plan.fetch_missing && may_ask`, each noted missing first (W:149-164); subscriptions, for a page only ids found held (W:166-175); a node run also notes missing every subscribed id it did not read this run (W:181-187); duty charged for messages and bytes (W:191-194).
- `scan` (W:323-417): state from `env.contract_state` (never the triggering message); absent, over 16 MiB or undecodable JSON is `Err` (unknown). Reads 16 tombstone shards, Kept, Archived and Flags indexes; any unreadable one aborts. Message id = `blake3(ciphertext)[..16]` (W:310). Skip if tombstoned or kept/archived; flags `deleted:true` skip, unreadable flags skip and count `remaining`; undecryptable is tombstoned (W:375-379); flags `{"read":false,"deleted":false}` written first, create-only (W:307, W:385-402); then the `wire-v1` `KeptMessage` create-only, with `signature_valid` recorded not enforced (W:403-414, W:439-452).
- `note_seen` (W:73): a full shard drops its lowest id.

### sealing.rs (169 lines): the v2 ciphertext
- `nonce24 || ML-KEM-768 ct (1088) || XChaCha20-Poly1305(len u32 BE || CBOR(SealedEnvelope) || zeros)`, `OVERHEAD 1128` (S:26-30), padded to the smallest of `CIPHERTEXT_SIZES` = 8, 16, 32 KiB (S:87-90, PW:42).
- Key `HKDF-SHA256(no salt, KEM secret, info "freenet-mail v2 message key")` (S:33-43); AAD `"fmail/ct/v2" || recipient address` (S:35, S:45).
- Sender signature: ML-DSA with context `fmail/v2/sealed` over `inner_binding(recipient, blake3(body), sent_time, kind, tag)` (S:79-86): it covers the body hash, never the ciphertext. `kind_and_tag` (S:50-59): pass id, empty for owner, the envelope's `ghost_vk` for a quarantine message that carries a proof.
- `open` (S:121-169) returns `None` unless size, KEM, AEAD, zero padding, decode, `message_id == id`, signature and `reply_pass.cert.recipient == address_of(sender_vk)` all hold.
- KEM encapsulation and nonce draw from the caller's `MixRng` (S:95-96).

### inbox_v2.rs (2105 lines)
- Numbers: lifetime 335 to 395 d (V:53-54), invite 14 d (V:55), `TIME_STEP_MAX 7 d`, `TIME_LEAD_MAX 1 d` (V:57-58), `EPOCH_GRACE 14 d` (V:60), `BUMP_AT_REVOCATIONS 2048` (V:62), `DRAIN_BATCH 32` (V:65), `STRANGERS_KEPT 1024` (V:70), `RESEND_AFTER 6 h` (V:73), `NO_PASS_RETRY 1 d` (V:76), `CLOCK_LAG 12 h` (V:78), `MAX_INVITES 1024` (V:81), `PRUNE_PER_CALL 64` (V:83), `RESEAL_MARGIN 6 h` (V:85), `HELD_PER_RECIPIENT 4` (V:1066).
- Keys from the ML-DSA seed, never stored: issuer Ed25519 = `blake3::derive_key("freenet-mail issuer key v1", seed)` (V:89); pass id and pass key from seed, contact and a ledger nonce (V:93-111). `page_rng` refuses all-zero entropy and binds the request (V:143-164); `background_rng` mixes host randomness with `H(seed, now, context)` (V:124-137).
- `create_inbox` (V:458-542): `Conflict` if a record exists; fresh = version 1, epoch 1, `Open`; with `existing` the state must verify (`own_state`, V:441-455: owner signature, time signature, same issuer key), version = existing + 1, tombstones and filters copied, `inherited_revoked` = highest epoch and expiry of the existing revocations. Record saved create-only before the reply.
- `republish_inbox` (V:557-605): next version, guarded time, tombstones in window, revocations re-signed from the ledger (`live_revocations` V:639-675 fails `Unreadable` on a listed entry it cannot read); an emergency bump when an inherited revocation may still be live (V:566-580). Returns a state with empty `usage` and `messages`.
- `guarded_time(prev, now) = now.min(min(prev+7d, now+1d)).max(prev)` (V:377-382).
- Passes: `start_term` (V:705) whole-day expiry, renewal at 40 to 60%; `reissue_at` (V:720) a per-pass point of the grace; `contact_pass` (V:885) reuses the live pass (refreshing in place, same id and key) or creates one, promoting a stranger; `issue_pass` (V:1008) marks a contact pass delivered (the page hands it over) or makes an invite. `prunable`/`prune_ledger` (V:953-1006).
- Held passes: `hold` (V:1092-1135) checks key against certificate, checks the certificate against the recipient's issuer only if this node holds that inbox, keeps 4 per recipient. `keep_drained_pass` (V:1196) applies the stranger budget to held passes.
- `drain` (V:1272-1322), `proven_ghosts` (V:1329), `store_inbound` (V:1407-1456: create-only; a `Conflict` is trusted only after `stored_copy` reads the same message back; evicts earliest stranger copies on `CapReached`), `store_opened` (V:1459-1522: invite use counted only when `issued_here` re-verifies).
- `owner_update` (V:1603-1836), the one path for `UpdateInboxV2` (limit 32, `from_node=false`, V:1593) and the jobs: `note_foreign`, drain, tombstones = record's plus state's in window (V:1625-1642), stranger cap (V:1646-1659), change, then **record saved before the delta is returned** (V:1803). Outcomes `Signed | WaitsForDrain | WaitsForStorage` (V:1684-1697). `AdvanceTime` never passes the last time that keeps an undrained message admitted (V:1700-1729). A recorded time whose update was lost is sent again (V:1772).
- `revoke` (V:1542-1578: needs the ledger entry, signs `(pass id, max_expiry, current epoch)`), `bump` (V:1524-1536: `prev_epoch_until = max(owner_time, now) + 14 d`, 0 for an emergency; every contact pass is then scheduled at its own `reissue_at`, V:1758-1769).
- `note_sent` (V:820-851) records the message that carries a pass's newest certificate as `pending` and schedules the read-back; `seal_v2` calls it for mail, the renew job with `resend`.
- Key-only restore is `ImportIdentity` from the recovery key followed by `CreateInboxV2` with `existing` (T/inbox_jobs.rs:1035-1131): the record is rebuilt from the network state, while the ledger, held passes and drained mail are not recoverable from the key.
- Sender-side checks mirror the contract: `sent_time_for` (V:1869, hour steps clamped into the recipient's window), `pass_room` (V:1878-1906: issuer signature, epoch, expiry, revocation, invite use), `queue_full` (V:1909-1928).
- `seal_v2` (V:1931) and `seal_to` (V:1961-2105): verify the recipient state (`their_state` V:1842: owner and time only), pick the admitted held pass with the highest `(epoch, expiry, room)`, else `Owner` for a self-send, else quarantine; always attach the sender's own contact pass as `reply_pass` (so the sender must have a v2 record, V:892); `pass_grant` always `None` (V:2017); notices `PassLapsed`, `PassFull`, `QueueFull` are not errors.
- `reseal_at = max(sent_time + WINDOW_BACK - (TIME_LEAD_MAX + skew + RESEAL_MARGIN), now + RESEAL_MARGIN)`, `skew = owner_time - now` (V:2090-2094): 18 hours after `sent_time`, less the recipient's observed lead.

### inbox_jobs.rs (557 lines)
- Numbers: `MAX_INBOXES_PER_RUN 4` (IJ:40), `OWNER_SLOTS 1` (IJ:48), `RENEW_PER_RUN 4` (IJ:56), `RENEW_LOOKS_PER_RUN 16` (IJ:60), `RENEW_COST_MS 120` (IJ:77), backoff 1 h doubling to a day (IJ:44-45, IJ:553), page cursor `bg/cursor-v2` (IJ:79).
- `code_hash()` is `include_str!` of `published-contract/candidate/inbox_v2_code_hash` (IJ:83-85); `inbox_id` derives every v2 contract id from it, for contacts under *our* `ghost_master` (IJ:88-96, IJ:120).
- `own` (IJ:109): contacts due within 6 h become `Tier::Contact` subscription candidates (IJ:143).
- `run` (IJ:177-260): up to 4 inboxes from the rotation; a node run pays 10 ms per inbox and stops the loop when duty is out (IJ:217-219). `drain_own` (IJ:285-348): state from the node's store, `AdvanceTime` when `now >= owner_time + 12 h`, else `Drain` if a message is untombstoned and a slot is left; `may_write` checked; rotation saved as the work begins (IJ:264-277, IJ:332); `limit = min(left-1, 32)`, `from_node=true`; update sent as `UpdateData::Delta`.
- `Renewing::run`/`one` (IJ:365-549): due contacts oldest first; ledger entry absent = backoff counted in the record, never "no pass" (IJ:450-455); read back `w::delivered` (IJ:476-482); **a pass never seen delivered is never pushed** (IJ:490-500); `seal_to(PassUpdate)`, sent only if the admission is `Pass`, else retry in a day (IJ:533-538); next look in 6 h.

## 3. Interfaces and state

- **Ops** (P:370-434): `RunJobs`, `CreateInboxV2{addr, ghost_master, existing}`, `UpdateInboxV2{addr, state, change, ui_entropy}`, `IssuePass{addr, contact, invite, ui_entropy}`, `HoldPass{addr, pass}`, `SealV2{from, to, to_state, content, ghost_vk, ui_entropy, reseals}`, `RepublishInboxV2{addr}`. `MAX_RESEALS 4` (P:438). Replies `JobsRun{scanned, imported, unknown, remaining}` (P:637), `InboxV2State`, `InboxV2Update` (P:466-487), `PassIssued`, `SealedV2` (P:522-536).
- **Wire** (PW): `WINDOW_BACK 2 d` (PW:34), `WINDOW_FORWARD 400 d`, `QUARANTINE_FORWARD 2 d`, hour `sent_time` (PW:36-39), `PER_PASS 8`, `PER_INVITE 2`, `QUARANTINE_SLOTS 16` (PW:44-46), `MAX_REVOKED = MAX_TOMBSTONES = 4096` (PW:48-49), signed layouts PW:194-332.
- **State per identity for v2** (all via `store::Space`, compare-and-swap):
  - `ItemKind::InboxV2` (one, id empty), `InboxRecord` (V:180-245): `ghost_master, max_version, max_owner_time, pass_epoch, prev_epoch_until, quarantine, epoch_began, reissue_grace, reissue_seed, issued (contact to pass id), due (renew schedule), strangers, stranger_seq, strangers_dropped, stranger_held, tombstones, invites, pruned_to, absent_looks, foreign_seen{version, at}, signed_owner (digest), inherited_revoked{epoch, expiry}`.
  - `PassLedger` (id = pass id, cap 4096), `LedgerEntry` (V:298-326): `issued_for, contact, nonce, class, epoch, expiry, max_expiry, renew_at, revoked, drained (invite uses), delivered, pending{id, expiry, at}, revoked_epoch, misses`.
  - `HeldPass` (id = recipient address, cap 2048): `Vec<Pass>` with secret pass keys.
  - `Inbound` (id = message id, cap 4096): `InboundMessage{id, sender_vk, sent_time, admission kind, ghost_vk, body}`.
  - Page access `(read, write, delete)` (ST:214-221): `InboxV2` and `PassLedger` read only; `HeldPass` nothing; `Inbound` read and delete. Caps ST:233-236.
- **Shared job state**: `bg/budget`, `ix/bg/origins` (all-zero origin); `bg/cursor`, `bg/cursor-v2` per origin; v1 tombstones `ix/sn/<owner>/<nibble>` (ST:248-260).
- **Host**: `Env` = secrets, `host_random`, `now_secs`, `contract_state` (local only), `log` (`delegates/mail-delegate/src/env.rs:15-30`); outbound Get, Subscribe, Update(Delta) requests. No feature flag gates anything in these five files.

## 4. Invariants and where they are held

| Rule | Code | Test / guard |
|---|---|---|
| Inbox read from the node's store; a notification's payload is never used | W:331-336, H:57-81 | T/jobs.rs:712, :724, :739, :761 |
| No answer is Unknown, never empty | W:316, W:331-338 | T/jobs.rs:629, :650, :660, :694 |
| v1 import create-only; deleted stays gone; tombstone before delete | W:356-414, H:1292-1306 | T/jobs.rs:817, :832, :846, :945, :1086 |
| Unread flags first, existing flags stand | W:385-402 | T/jobs.rs:912, :926, :1291 |
| Budget and rotation saved before work | W:119-124, IJ:264-277 | T/jobs.rs:1341; T/inbox_jobs.rs:2425 |
| A page run asks the network for nothing | W:155, W:169 | T/jobs.rs:547, :1532 |
| Version and time recorded before they leave | V:527-529, V:594-596, V:1803 | T/inbox_v2.rs:418; G:341, G:359 |
| Tombstone only after stored | V:1296-1320, V:1671-1697 | T/inbox_v2.rs:651, :694, :707; T/inbox_jobs.rs:377, :696; G:333 |
| Decision 25 note is never cleared | V:269-285, V:1616 | T/inbox_jobs.rs:2624, :2653, :2677; T/inbox_v2.rs:1219, :1242 |
| Binding names the message, own ML-DSA context | S:79-86, S:146-160 | T/inbox_v2.rs:727, :754; T/inbox_v2_r1.rs:144-288 |
| Ghost Key kept only with a checked proof | V:1301-1305, V:1329-1353, V:1494 | T/inbox_v2_r1.rs:314, :667; T/inbox_jobs.rs:2332 |
| Renewal never through quarantine, never for an undelivered pass | IJ:490-500, IJ:533-538 | T/inbox_jobs.rs:211, :246 |
| Strangers keep the newest 1024, contacts never evicted | V:1444-1448, V:1646-1659 | T/inbox_jobs.rs:810, :1751, :1821; T/inbox_v2_r1.rs:450 |
| Page cannot write records or read held passes | ST:214-221 | T/inbox_v2.rs:487 |
| Per-run and per-minute limits | J:330-382, IJ:210, IJ:329 | T/inbox_jobs.rs:1160-1403 |
| Bumps at 2048 live revocations and at a full set | V:1560-1564 | T/inbox_bump.rs:69, :110 |
| Owner version never wraps; page state must verify | V:429-455 | T/inbox_v2_r1.rs:626 |
| Manifest tags, import allowlist | lib.rs:42-46, J:82-87 | T/jobs.rs:173 (reads the committed candidate WASM), :203 |

## 5. State

Nothing here is frozen. The delegate ships as `published-contract/candidate/mail_delegate` (built with `v1-transition`), to be frozen into `published-contract/wasm/v2/` at release; the inbox v2 contract is a candidate too and its code hash is compiled into the delegate (IJ:83-85). Once shipped, the derive-key strings (V:90, V:94, V:103), HKDF info and AAD (S:33-35), the record shapes and every signed layout become permanent. `InboxRecord.v` and `LedgerEntry.v` are written as 1 and never checked on load (V:336-351).

## 6. Tests

All native, `MemEnv` as the host (`T/jobs.rs:1-6`); the v2 files run every state and delta through the contract crate's `apply`/`is_valid` with `Rules::V1` via `World` (`T/inbox_v2.rs:28-261`).
- `T/jobs.rs` (78 tests): planner arithmetic (ranking, budgets, rotation, triggers) and the v1 watch through `handlers::process`.
- `T/inbox_v2.rs` (21): full round, time guard, lost update, RequireGhost, page access, drain fault sweep (`DRAIN_SEEDS` 60, `DRAIN_RATE` 60, `DRAIN_BLIND` 0, `DRAIN_FROM`; :653-658), republish, decision 25 note.
- `T/inbox_v2_r1.rs` (13): signature domains against v1 signers, unproven Ghost Key, held-pass choice and caps, zero entropy.
- `T/inbox_jobs.rs` (47): `watch::run` with updates applied by the contract (`run`, `settle`, :44-86): renew, clock, drain sweep (`JOB_DRAIN_SEEDS` 40, `_RATE`, `_BLIND`, `_FROM`; :379-382), time sweep (`JOB_TIME_SEEDS` 6, :655-658), bump reissue, reseal, strangers (`STRANGER_EXTRA`), Ghost Key fixtures, key-only restore (:1035), budgets, write counts (:1475), takeover and decision 25.
- `T/inbox_bump.rs` (2), `T/inbox_vectors.rs` (3, against `contracts/inbox-v2/tests/fixtures/vectors.txt`).
- Run: `cd delegates/mail-delegate && cargo test --features v1-transition` (what `scripts/check-excluded-crates.sh test` does, :30; also `cargo make test`). One module: append `-- tests::inbox_jobs::`. Needs the committed candidate WASM and the contract's fixtures; dependencies build at `opt-level 2`.
- Guards (`G`): 66 mutate inbox_v2.rs, 22 inbox_jobs.rs, 3 sealing.rs, 2 jobs.rs, **0 watch.rs**. Real-node proofs: `cargo make test-e2e-background`, `cargo make test-e2e-inbox-v2` (tests-qa map).

## 7. Changing it safely

- Any edit under the delegate, `protocol/` or `wire-v1/` (tests included) needs `published-contract/candidate/mail_delegate` and its code hash rebuilt (`published-contract/candidate/README.md:18-36`), on linux/amd64 or from the CI artifact.
- **A change to `contracts/inbox-v2` changes `inbox_v2_code_hash`, which the delegate (IJ:84) and the UI (`ui/src/v2/mod.rs:29`) embed: rebuild the delegate candidate after the contract's.**
- Guards anchor on exact source text that must occur once (G:286-294): reformatting or moving an anchored line makes the guard fail as "mutation not applied".
- Wake-up changes must keep lib.rs:45 and `Tick::ALL` in step; new host imports must be in `scripts/delegate-imports-allowlist.txt`; never add `getrandom`/`rand`.
- New fields in `InboxRecord`, `LedgerEntry`, `Budget` need `#[serde(default)]`; protocol encodings are pinned in `protocol/src/golden_v1.txt`.
- Cost constants (J:37-54, IJ:77) are estimates against core's limits; the write-count test (T/inbox_jobs.rs:1475) is the only automatic check.
- Traps: `owner_update` saves the record once at the end, after ledger and held-pass writes; `set_due`, `note_absent`, `note_sent` each rewrite the whole record; `seal_to` issues the reply pass before sealing can fail.

## 8. Docs versus code

1. AGENTS.md "Queued sends": `reseal_at` is "two days after the earlier of its `sent_time` and the sealing time". Code: `sent_time + 18 h - skew`, never before `now + 6 h` (V:2090-2094); pinned by T/inbox_jobs.rs:784-785.
2. `published-contract/candidate/README.md:15` says of inbox v2 "Nothing embeds it yet", and it and AGENTS.md list only the delegate, `protocol/`, `wire-v1/` as reasons to rebuild the delegate. The code hash is embedded (IJ:84).
3. AGENTS.md and H:82: "every refresh asks for every subscription again". A node run drops ids under backoff (J:454-456) and puts under backoff every subscribed id it did not read in that run (W:181-187); on the hourly `v1watch` tick no v2 inbox is read, so each one is noted and waits for a rescan to clear it. T/jobs.rs:1429 pins the zero-subscription window.
4. V:56 and T/inbox_v2.rs:398: `owner_time` "never past node time + 1 d". `guarded_time` never signs past `now` itself (V:381; test asserts `guarded_time(100,120) == 120`).
5. IJ:52-55: a day's 384 renewals "cost under 2% of the day's duty". 384 x 120 ms = 46 s against 864 s of daily duty, about 5%.
6. AGENTS.md: only "a run that no page started" lacks page entropy. A page's `RunJobs` uses `background_rng` too (IJ:333, IJ:520); `Op::RunJobs` carries none (P:370).
7. AGENTS.md and V:8-11 list the record as version, time, epoch, filters, contact to pass id. It also holds up to 4096 tombstones, the stranger maps, invites and the schedule (V:196-244).
8. `JobsRun.imported` is documented as "stored as kept copies" (P:640); it also counts v2 drains (IJ:338).
9. AGENTS.md: renewals go "whether or not the user writes to that contact". Only for a pass seen delivered once (IJ:490); `IssuePass` marks that (V:1024-1033).

## 9. Suspected weak spots

1. **Record size** (medium on arithmetic, low on reachability): `InboxRecord` is one item capped at `MAX_ITEM_BYTES` 256 KiB (P:30, H:1159). 4096 tombstones (about 160 KB), 1024 strangers, 1024 `stranger_held`, 1024 invites plus `issued`/`due` per contact (about 75 bytes each) can exceed it; `save` then fails and drains and sends for that identity stop. No test measures it.
2. **Republish drops usage** (low to medium): the republished state has `usage` empty (V:601), so an exhausted invite admits again until the next drain re-publishes its count.
3. **Self-send** (medium): `seal_to` issues a contact pass to oneself (V:1975-2010) and schedules it; its renewal can only be `Owner`, so after the first renewal point the job seals and discards a message daily (IJ:536-538). No delegate test seals with `to == from`.
4. Unreadable bookkeeping is treated as "not held": GET sent and backoff noted for a held inbox (W:135-140, W:149-163). High confidence, low impact.
5. A full `missing` list always overwrites entry 0 (J:320). High, low impact.
6. Every run, a forged notification included, lists all origins and decodes every identity's `InboxRecord` before any budget check (W:204-253, IJ:109-128). Cost only.
7. A message stored but not tombstoned (tombstones full) is counted as drained again each run (V:1428-1442, V:1505).
8. `revoke` and the bump thresholds trust the page state's unverified `revoked` set (V:1555-1564, V:1842-1865).
9. `now = 0` when the host gives no time resets the budget and backoffs (W:90, J:279-285).

## 10. Open work and documented limits

- `Job::SendQueued` is listed for `RunJobs` and nothing implements it (J:131-132, J:153).
- `SealedBody.pass_grant` is read by the drain (V:1479-1483) but never produced or tested.
- `Candidate.activity` is always 0 (IJ:150), so "recent contacts first" never applies.
- No unsubscribe: a deleted identity's subscription stays (AGENTS.md).
- An unreadable tombstone shard stops that identity's v1 watch (W:341-344).
- Origins past 16 are served only by their own `RunJobs` (W:54-56).
- The wasm timing harness behind `RENEW_COST_MS` is not in the repo (IJ:73-76).
- After a takeover, a later `RepublishInboxV2` forces an emergency bump while inherited revocations may be live (V:566-580); passes issued before a restore cannot be revoked one by one (V:1552-1553).
- `v1-transition` is to be removed with the v1 send path.

## 11. Questions

- Core's limits quoted in comments (5 s a call, 90 s network wait, 8 queued calls, `BudgetLimits`) are not in this tree.
- Does core fetch a contract on Subscribe when it holds no state (W:176-178)?
- Where are the numbered decisions (6, 9, 17 to 25) recorded? Only AGENTS.md and `docs/codebase-notes/inbox-v2-contract.md` cite them.
- What is the "90-day window" for v1 inboxes (J:179-180)? No code implements it.
- Is the self-send pass intended?

---

## Appendix: the reader's structured summary

Five files of the out-of-workspace mail delegate: everything it does with no page open, and every inbox v2 step that needs a secret.

jobs.rs is a pure planner (trigger to job table, duty, ops and write budgets, scan rotation, missing-inbox backoff, subscription ranking). watch.rs is the single `run` every trigger reaches (NodeStarted, four wake-ups, contract notifications, GET answers, a page's RunJobs); it also holds the v1 watch, which reads v0.2.x inboxes from the node's store and imports create-only kept copies with unread flags and tombstones. inbox_v2.rs holds the owner side (create, takeover, republish, drain-then-sign `owner_update`, passes, invites, revocation, epoch bumps, held passes, stranger cap, the "another node" note) and the sender side (`seal_to`, admission choice, `reseal_at`). sealing.rs is the ML-KEM-768 plus XChaCha20-Poly1305 envelope with an ML-DSA binding over the body hash. inbox_jobs.rs runs drain, clock and renew from `watch::run`.

State: nothing is frozen. The delegate is a committed candidate WASM that must be rebuilt after any source change. The code is heavily tested natively, with every v2 state and delta run through the contract crate, three fault sweeps and about 93 mutation guards (none mutates watch.rs).

I found one clear AGENTS.md error (`reseal_at` is 18 hours after `sent_time`, not two days), one undocumented rebuild dependency (the delegate embeds the inbox v2 candidate's code hash), and a few weak spots, none confirmed by running anything (read-only). Unfinished: `Job::SendQueued`, `pass_grant`, contact activity ranking.

The map file is complete on disk, so this unit need not be redone after a restart.

### Key facts

- Every trigger goes through watch::run: host messages via handlers::background (delegates/mail-delegate/src/handlers.rs:62-81, :97-101), a page's RunJobs via run_jobs (handlers.rs:324-332). Subscribe and Update answers are dropped unread (handlers.rs:85-88).
- Manifest wake-ups are rescan=900 s, v1watch=3600 s, renew=21600 s, clock=86400 s (delegates/mail-delegate/src/lib.rs:42-46); their tags must match Tick::ALL (delegates/mail-delegate/src/jobs.rs:82-87). A test reads the manifest from the committed candidate WASM (delegates/mail-delegate/src/tests/jobs.rs:173).
- Trigger to jobs table is jobs_for (jobs.rs:137-164): only NodeStarted, RunJobs and Tick(V1Watch) carry V1Refresh, so only they subscribe and set fetch_missing (jobs.rs:465); Tick(Rescan) scans, drains, moves the clock and renews; Contract(id) scans, drains and renews without the clock.
- Planner numbers: 4 v1 scans and 12 messages a run, duty 10 ms/s up to 10 s, SCAN_COST_MS 10, MESSAGE_COST_MS 120, 300 ops a minute, 60 updates per contract a minute, 256 subscriptions, missing-inbox backoff 1 h doubling to 1 day (jobs.rs:21-70).
- watch::run saves the budget and rotation before any import (delegates/mail-delegate/src/watch.rs:119-124); a page run uses its own cursors bg/cursor and bg/cursor-v2 in its origin namespace (watch.rs:99-118, delegates/mail-delegate/src/inbox_jobs.rs:264-277) and never sends a GET (watch.rs:155) or subscribes an inbox it did not find held (watch.rs:169).
- The v1 watch reads the inbox only from env.contract_state (watch.rs:331-336); it skips tombstoned, kept, archived or deleted-flagged messages, writes flags {"read":false,"deleted":false} first and then the kept copy, both create-only (watch.rs:356-414). v1 tombstones are 16 shards of 512 per identity (watch.rs:58, delegates/mail-delegate/src/store.rs:248-260).
- Per identity the delegate keeps for v2: one InboxRecord (ItemKind::InboxV2; delegates/mail-delegate/src/inbox_v2.rs:180-245), one LedgerEntry per issued pass (PassLedger; inbox_v2.rs:298-326), up to 4 held passes per recipient (HeldPass; inbox_v2.rs:1066, :1114-1134) and drained mail (Inbound; inbox_v2.rs:1495-1503). A page may read InboxV2 and PassLedger, read and delete Inbound, and do nothing with HeldPass (store.rs:214-221).
- The InboxRecord holds far more than version, time, epoch and filters: issued and due maps, strangers, stranger_held, up to 4096 tombstones, invites, absent_looks, foreign_seen, signed_owner, inherited_revoked (inbox_v2.rs:196-244). It is one item, rewritten whole by owner_update, set_due, note_absent and contact_pass.
- owner_update is the single path for UpdateInboxV2 and the jobs (inbox_v2.rs:1603-1836): note_foreign, drain, merge tombstones, stranger cap, apply the change, save the record with compare-and-swap (inbox_v2.rs:1803), and only then return the delta. Outcomes are Signed, WaitsForDrain, WaitsForStorage (inbox_v2.rs:1684-1697).
- All v2 keys derive from the ML-DSA seed and are never stored: issuer key (inbox_v2.rs:89-91), pass id and pass key from seed, contact and a ledger nonce (inbox_v2.rs:93-111). Changing those derive-key strings or the HKDF info and AAD (delegates/mail-delegate/src/sealing.rs:33-35) breaks every inbox and message.
- Ciphertext is nonce24, ML-KEM-768 ct (1088 bytes), then XChaCha20-Poly1305 over a length-prefixed CBOR envelope zero-padded to 8, 16 or 32 KiB (sealing.rs:26-30, :87-94; protocol/src/inbox_v2.rs:42); the sender's ML-DSA signature covers inner_binding over blake3(body), not the ciphertext (sealing.rs:79-86).
- reseal_at = max(sent_time + WINDOW_BACK - (TIME_LEAD_MAX + skew + RESEAL_MARGIN), now + RESEAL_MARGIN) (inbox_v2.rs:2090-2094); with WINDOW_BACK 2 d (protocol/src/inbox_v2.rs:34) that is 18 h after sent_time. SealV2 refuses reseals above MAX_RESEALS 4 (protocol/src/lib.rs:438, inbox_v2.rs:1941).
- seal_to always issues or refreshes the sender's own contact pass as reply_pass (inbox_v2.rs:2010), so SealV2 fails NotFound for a sender with no v2 record (inbox_v2.rs:892), and a self-send issues a pass to oneself.
- The renew job never pushes a pass that was never seen delivered (inbox_jobs.rs:490-500) and never sends through quarantine (inbox_jobs.rs:533-538); per run it has 16 looks and 4 sends (inbox_jobs.rs:56-60, :210), each send charged RENEW_COST_MS 120 (inbox_jobs.rs:77).
- The clock job is owner_update with AdvanceTime once now >= owner_time + 12 h (inbox_jobs.rs:320; CLOCK_LAG inbox_v2.rs:78); guarded_time moves at most 7 days a step and never past the node's own clock (inbox_v2.rs:377-382).
- The delegate compiles in the inbox v2 candidate's code hash via include_str! (inbox_jobs.rs:83-85), and every v2 contract id it computes depends on it (inbox_jobs.rs:88-96). A contact's inbox id is computed with our own ghost_master (inbox_jobs.rs:120).
- Decision 25: note_foreign runs before anything is signed and nothing clears foreign_seen (inbox_v2.rs:269-285, :1616); signed_owner is the digest that tells this node's record from another's at the same version (inbox_v2.rs:262-264).
- Takeover (CreateInboxV2 with existing) copies tombstones and filters and records only the highest epoch and expiry of inherited revocations (inbox_v2.rs:500-523); a later RepublishInboxV2 forces an emergency epoch bump while those may be live (inbox_v2.rs:566-580).
- Tests run natively with `cd delegates/mail-delegate && cargo test --features v1-transition` (scripts/check-excluded-crates.sh:30). Sweep knobs: DRAIN_* (delegates/mail-delegate/src/tests/inbox_v2.rs:653-658), JOB_DRAIN_* and JOB_TIME_* (delegates/mail-delegate/src/tests/inbox_jobs.rs:379-382, :655-658).
- Mutation guards anchor on exact source text that must occur once (scripts/check-guards-fail.sh:286-294): 66 edit inbox_v2.rs, 22 inbox_jobs.rs, 3 sealing.rs, 2 jobs.rs, none watch.rs. Reformatting an anchored line breaks its guard.

### Docs or comments that disagree with the code

- **[medium]** Claim (AGENTS.md, mail delegate section, bullet 'Queued sends (decision 9)'): SealedV2.reseal_at is two days after the earlier of its sent_time and the sealing time. **Reality:** reseal_at = max(sent_time + WINDOW_BACK - (TIME_LEAD_MAX + skew + RESEAL_MARGIN), now + RESEAL_MARGIN): 18 hours after sent_time less the recipient's observed clock lead, never sooner than 6 hours after sealing. A test pins sent_time + WINDOW_BACK - (TIME_LEAD_MAX + RESEAL_MARGIN). Evidence: delegates/mail-delegate/src/inbox_v2.rs:2090-2094; delegates/mail-delegate/src/tests/inbox_jobs.rs:784-785
- **[medium]** Claim (published-contract/candidate/README.md:15 and :18-21; AGENTS.md paragraph after the mail delegate bullets ('Rebuild it after changing the delegate, protocol/ or wire-v1/')): Nothing embeds the inbox v2 candidate yet; rebuild the delegate candidate after changing the delegate, protocol/ or wire-v1/. **Reality:** The delegate and the UI embed the candidate's code hash with include_str!, so a contract change that moves inbox_v2_code_hash changes the delegate's bytes and forces a delegate candidate rebuild too. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:83-85; ui/src/v2/mod.rs:29
- **[low]** Claim (AGENTS.md 'Background jobs' bullet; code comment delegates/mail-delegate/src/handlers.rs:82-84): A subscription answer is ignored and every refresh asks for every subscription again. **Reality:** A node run drops from the subscribe list every id under backoff, and puts under backoff every subscribed id it did not read in that same run, including held inboxes outside the scan slice and every v2 inbox on the hourly v1watch tick (which runs no v2 job). Those are re-subscribed only after a later run reads them or the backoff ends. Evidence: delegates/mail-delegate/src/jobs.rs:454-456; delegates/mail-delegate/src/watch.rs:181-187; delegates/mail-delegate/src/tests/jobs.rs:1429-1449
- **[low]** Claim (Code comment delegates/mail-delegate/src/inbox_v2.rs:56 and test doc delegates/mail-delegate/src/tests/inbox_v2.rs:398-399): owner_time never moves past the node's clock plus a day. **Reality:** guarded_time is now.min(min(prev+7d, now+1d)).max(prev), which never exceeds now unless prev already did; the one-day lead never applies to this delegate's own signing and is used only as an assumption in reseal_at. Evidence: delegates/mail-delegate/src/inbox_v2.rs:377-382; delegates/mail-delegate/src/tests/inbox_v2.rs:411
- **[low]** Claim (Code comment delegates/mail-delegate/src/inbox_jobs.rs:49-55): A day's renewals (384) cost under 2% of the day's duty. **Reality:** 384 renewals at RENEW_COST_MS 120 are 46 s; the duty refill of 10 ms per second is 864 s a day, so about 5% (more with the 10 ms look charge). Evidence: delegates/mail-delegate/src/inbox_jobs.rs:77; delegates/mail-delegate/src/jobs.rs:27
- **[low]** Claim (AGENTS.md 'Inbox v2 jobs' bullet, last sentence): Only a run that no page started has no page entropy. **Reality:** A page's RunJobs uses background_rng for its drains and renewals as well; Op::RunJobs carries no entropy. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:333 and :520; protocol/src/lib.rs:370
- **[low]** Claim (AGENTS.md 'Inbox v2' bullet under the mail delegate; module doc delegates/mail-delegate/src/inbox_v2.rs:8-11): The inbox record keeps the highest owner version and owner_time signed, epoch, filters and contact to pass id. **Reality:** Incomplete: the record also holds the renewal schedule, stranger bookkeeping, up to 4096 tombstones, the invite set, prune cursor, absent-look counts, foreign_seen, signed_owner and inherited_revoked. Evidence: delegates/mail-delegate/src/inbox_v2.rs:196-244
- **[low]** Claim (protocol/src/lib.rs:640-641 (doc comment)): JobsRun.imported is 'Messages stored as kept copies'. **Reality:** It also counts inbox v2 messages drained (AGENTS.md says so for the log line), and a stored message that could not be tombstoned is counted again on each later run. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:338; delegates/mail-delegate/src/inbox_v2.rs:1505
- **[low]** Claim (AGENTS.md 'Inbox v2 jobs' bullet): The renew job sends each contact pass's certificate whether or not the user writes to that contact. **Reality:** Only for a pass seen delivered at least once (IssuePass marks it delivered; mail must be read back). A pass that only rode on mail never seen to land is read back for two days and then dropped from the schedule, never pushed. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:490-500; delegates/mail-delegate/src/inbox_v2.rs:1024-1033
- **[low]** Claim (Code comment delegates/mail-delegate/src/jobs.rs:131-132): Job::SendQueued: 'Hand a page the queued work the delegate cannot do itself.' **Reality:** The job is listed for RunJobs but no code in watch.rs or inbox_jobs.rs acts on it, and JobsRun has no field for queued work. Evidence: delegates/mail-delegate/src/jobs.rs:153; protocol/src/lib.rs:637-647

### Suspected issues (leads from reading, not confirmed)

- **InboxRecord may outgrow the 256 KiB item limit** (confidence: medium). The record is a single item written through put_item, which refuses bodies over MAX_ITEM_BYTES. Its maps are bounded only individually: 4096 tombstones (about 160 KB encoded), 1024 strangers, 1024 stranger_held, 1024 invites, and issued/due per contact (about 75 bytes each, up to the 4096-entry ledger cap). By my estimate the sum can pass 256 KiB; save would then fail with BadInput and owner updates and sends for that identity would stop (mail stays in the inbox). No test measures the encoded size. Not reproduced. Evidence: delegates/mail-delegate/src/inbox_v2.rs:196-220; protocol/src/lib.rs:30; delegates/mail-delegate/src/handlers.rs:1159
- **RepublishInboxV2 returns no usage entries** (confidence: low). The republished state re-signs live revocations from the ledger but sets usage to an empty map, although the ledger holds each invite's drained set. An invite already used up would be admitted again until the next drain of a message under it re-publishes its count. Depends on the contract's usage rule, which I did not re-read. Evidence: delegates/mail-delegate/src/inbox_v2.rs:597-603
- **Self-send issues a contact pass to oneself whose renewal can never be sent** (confidence: medium). seal_to calls contact_pass even when to == from, creating a ledger entry and a renewal schedule for the sender's own address. A renewal to oneself is sealed with the Owner admission, which the renew job treats as 'no pass' and retries every day, sealing and discarding a message each time once the first renewal point has passed. No delegate test seals with to == from. Evidence: delegates/mail-delegate/src/inbox_v2.rs:1975-2010; delegates/mail-delegate/src/inbox_jobs.rs:536-538
- **Unreadable bookkeeping is treated as an inbox not held** (confidence: high). Any Err from scan (an unreadable tombstone shard or index, not only an absent contract) puts the id in `missing`; a node run with fetch_missing then notes a backoff and sends a GET for an inbox the node holds, and the backoff suppresses its subscription refresh until a scan succeeds. Evidence: delegates/mail-delegate/src/watch.rs:135-140 and :149-163
- **A full missing-inbox list always overwrites its first entry** (confidence: high). With 256 backoff entries, each new one replaces index 0, so new missing inboxes evict one another and the other 255 are never displaced. Low impact. Evidence: delegates/mail-delegate/src/jobs.rs:316-321
- **Unbudgeted fixed cost on every run, including forged notifications** (confidence: medium). targets() lists every origin and identity and inbox_jobs::own decodes each identity's whole InboxRecord before any duty is checked or charged; a notification for a contract nobody watches still pays this. Each renewal look also rewrites the whole record through set_due. Cost concern only; not measured. Evidence: delegates/mail-delegate/src/watch.rs:204-253; delegates/mail-delegate/src/inbox_jobs.rs:109-128; delegates/mail-delegate/src/inbox_v2.rs:769-788
- **Revocation and bump thresholds trust an unverified part of the page's state** (confidence: medium). their_state/own_state verify only the owner record and the time. revoke counts state.revoked to decide a routine or emergency epoch bump, so a page can force a bump with a crafted revoked set. The page is the user's own origin, so impact is low. Evidence: delegates/mail-delegate/src/inbox_v2.rs:1555-1564 and :1842-1865
- **A missing host clock resets the job budget** (confidence: low). watch::run uses now = 0 when now_secs is None; Budget::advance then sees a clock more than a day back and starts over, losing the missing-inbox backoffs and write counts, and v1 imports get kept_at 0. On wasm the clock is None only for a negative timestamp, so this is unlikely. Evidence: delegates/mail-delegate/src/watch.rs:90; delegates/mail-delegate/src/jobs.rs:278-286; delegates/mail-delegate/src/env.rs:49-54
- **Record version fields are never checked** (confidence: high). InboxRecord.v and LedgerEntry.v are written as 1, but load decodes without looking at them, so a future shape change cannot be told apart by version unless that check is added. Evidence: delegates/mail-delegate/src/inbox_v2.rs:336-351 and :476, :863

### Open work

- Job::SendQueued is declared and listed for RunJobs but nothing implements it (delegates/mail-delegate/src/jobs.rs:131-132, :153).
- SealedBody.pass_grant is handled by the drain but never produced by seal_to and never tested with a value (delegates/mail-delegate/src/inbox_v2.rs:1479-1483, :2017).
- Candidate.activity is always 0 from inbox_jobs::candidates, so the 'recent contacts first' ranking never applies outside tests (delegates/mail-delegate/src/inbox_jobs.rs:146-151).
- The inbox v2 code hash is the candidate's 'until a release freezes it' (delegates/mail-delegate/src/inbox_jobs.rs:81-85); the delegate itself is a candidate, not frozen.
- No unsubscribe: a deleted identity's subscription stays until core has one (AGENTS.md documented limit; no unsubscribe request anywhere in delegates/mail-delegate/src/watch.rs).
- An unreadable v1 tombstone shard stops that identity's watch; any unreadable list aborts the scan (delegates/mail-delegate/src/watch.rs:341-350).
- v1 tombstones are bounded at 16 shards of 512, after which an old one's protection lapses (delegates/mail-delegate/src/watch.rs:26-28, :58, :79-83).
- Origins past 16 are worked for only by their own RunJobs (delegates/mail-delegate/src/watch.rs:54-56).
- The wasm timing harness behind RENEW_COST_MS is not kept in the repository; only write counts are tested (delegates/mail-delegate/src/inbox_jobs.rs:73-76; delegates/mail-delegate/src/tests/inbox_jobs.rs:1475).
- Passes issued before a key-only restore cannot be revoked individually (revoke needs the ledger entry: delegates/mail-delegate/src/inbox_v2.rs:1552-1553) and cannot be renewed.
- A stranger message whose tombstone gave way can return and be drained again (delegates/mail-delegate/src/inbox_v2.rs:1656-1658).
- watch.rs has no mutation guard in scripts/check-guards-fail.sh, and no delegate test covers a self-send (SendAdmission::Owner through seal_to).
- The v1-transition feature is to be removed with the v1 send path (delegates/mail-delegate/Cargo.toml:60-62).

### Open questions

- Core's limits quoted in comments and AGENTS.md (5 s per call, 90 s network wait, 8 queued page calls timing out at 30 s, BudgetLimits) are not in this tree: are they still what freenet-core does at the pinned stdlib 0.12.1?
- Does core fetch a contract when asked to subscribe to one it holds no state for, as delegates/mail-delegate/src/watch.rs:176-178 assumes?
- Where are the numbered design decisions (6, 9, 17 to 25, section 5.1) recorded? Only AGENTS.md and docs/codebase-notes/inbox-v2-contract.md cite them.
- What is the '90-day window' after which inbox v2 'stops offering' v1 inboxes (delegates/mail-delegate/src/jobs.rs:179-180)? No code in these files implements it.
- Is issuing a contact pass to oneself on a self-send intended, and should the renew job skip it?
- Should RepublishInboxV2 re-sign usage entries from the ledger, as it does revocations?
- Is the hourly v1watch tick meant to leave every v2 inbox under a one-hour subscription backoff until a rescan reads it, or should a held inbox be exempt?
- None of this was run: the task was read-only, so every suspected issue rests on reading the code.

### Coverage

Read in full: `delegates/mail-delegate/src/jobs.rs`, `delegates/mail-delegate/src/watch.rs`, `delegates/mail-delegate/src/sealing.rs`, `delegates/mail-delegate/src/inbox_v2.rs`, `delegates/mail-delegate/src/inbox_jobs.rs`, `delegates/mail-delegate/src/lib.rs`, `delegates/mail-delegate/Cargo.toml`

Skimmed: `delegates/mail-delegate/src/tests/jobs.rs (module doc, all 78 test names, about 6 bodies)`, `delegates/mail-delegate/src/tests/inbox_jobs.rs (module doc, all 47 test names and doc comments, helpers, restore and reseal bodies, sweep assertions)`, `delegates/mail-delegate/src/tests/inbox_v2.rs (module doc, World helper in full, all 21 test names and docs, guard and lost-update bodies)`, `delegates/mail-delegate/src/tests/inbox_v2_r1.rs (module doc, all 13 test names and docs, no bodies)`, `delegates/mail-delegate/src/tests/inbox_bump.rs (module doc and test names)`, `delegates/mail-delegate/src/tests/inbox_vectors.rs (module doc, test names and docs)`, `delegates/mail-delegate/src/handlers.rs (lines 40-145, 225-350, 1150-1165, 1280-1310 only)`, `delegates/mail-delegate/src/store.rs (lines 150-264 and the cap constants)`, `delegates/mail-delegate/src/env.rs (Env trait and DelegateCtx impl excerpts)`, `delegates/mail-delegate/src/crypto.rs (MixRng only)`, `protocol/src/lib.rs (ItemKind, v2 ops, replies, JobsRun)`, `protocol/src/inbox_v2.rs (constants and item list by grep)`, `scripts/check-guards-fail.sh (guard mechanism, delegate test runners, guard names by grep)`, `scripts/check-excluded-crates.sh (delegate entry)`, `published-contract/candidate/README.md`, `Makefile.toml (test tasks)`

Skipped: (none)
