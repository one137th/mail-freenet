> Notes v1, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Map: inbox v2 contract (`inbox-v2-contract`)

Read-only survey at upstream 2358b59. Nothing was built or run: every statement is from reading. Paths are relative to the repo root; `C/` abbreviates `contracts/inbox-v2/`.

## 1. Purpose and place

`freenet-email-inbox-v2` (`C/Cargo.toml:7-8`, v0.3.0) is the Freenet contract that will hold one identity's inbox: undrained ciphertexts plus the owner's signed record, a signed clock and pass bookkeeping (`C/src/lib.rs:1-11`). One contract instance per inbox: the id is `hash(wasm, params)` and params are the address and a Ghost Key master (`C/src/lib.rs:30-40`). It has no storage, no clock, no related contracts. It is a candidate, not frozen: the WASM sits in `published-contract/candidate/inbox_v2`, and `.release-freeze:1-11` blocks releases until v0.3.0. The crate is excluded from the root workspace with its own lockfile (`Cargo.toml:38-40`, `C/Cargo.toml:1-4`). Clients never link it: `protocol/src/inbox_v2.rs:1-9` re-implements types and layouts, the delegate uses it as a dev-dependency only (`delegates/mail-delegate/Cargo.toml:45-47`), the UI embeds the candidate bytes (`ui/src/v2/mod.rs:27-29`).

## 2. Walkthrough

### `C/src/lib.rs` (184 lines): contract interface
- `Params { addr:[u8;24], ghost_master:[u8;32] }`, wire `0x02 ‖ addr ‖ master`, exactly 57 bytes (`:31-61`).
- `is_valid(rules, params, state)`: decode, merge the state alone, require equality (`:70-78`). This is the fixed-point test; it clones the state.
- `apply(rules, params, state, Vec<UpdateData>)`: decodes the held state, pushes each `State`/`Delta`/`StateAndDelta` as a part, refuses any other variant with `InvalidUpdate` (`:81-108`). Empty delta bytes are a no-op (`:126-131`). `Parts.sent` sums counts over the caller's parts only (`:88-93,116-120`).
- `impl ContractInterface for Inbox` (`:137-184`): `validate_state` returns `Valid`/`Invalid`, never `RequestRelated`; bad params give `ContractError::Other`; merge refusals give `InvalidUpdateWithInfo{reason}` (`:63-67`); summarize/delta decode failures give `Deser`. Production always passes `Rules::V1`.
- `compile_error!` if `plausible-mutants` is built for wasm32 (`:19-20`).

### `C/src/types.rs` (273): state and canonical CBOR
- Decode bounds: `MAX_MESSAGES=88`, `MAX_REVOKED=4096`, `MAX_USAGE=1024`, `MAX_TOMBSTONES=4096`, `MAX_INPUT_BYTES=4 MiB+256 KiB` (`:28-35`). They are constants, not derived from `Rules`; `tests/bounds.rs:353-360` pins them to `Rules::V1` (88 = 64+8+16).
- `bounded` visitor refuses a map at the entry that passes N (`:69-101`); `CallCounts::add` sums the four counts per call (`:43-66`).
- `InboxState { owner, time, revoked: Map<B16,Revocation>, usage: Map<B16,PassUsage>, messages: Map<B32,Message> }` (`:120-129`); `InboxDelta` is the same with `owner`/`time` optional (`:134-143`). All five keys must be spelled (a missing `Option` would re-encode differently).
- `OwnerRecord { owner_vk, kem_ek, issuer_vk, pass_epoch:u32, prev_epoch_until:u64, version:u64, quarantine, tombstones: Map<B32,u64>, sig }` (`:158-176`); `QuarantineMode::{Open, RequireGhost{min_usd:Option<u32>}}` (`:179-182`).
- `TimeAttestation{owner_time,sig}` (`:185-189`); `Revocation{pass_id,expiry,epoch,sig}` (`:192-197`); `PassUsage{pass_id,used:u8,expiry,owner_version,sig}` (`:200-209`); `PassCert{recipient,pass_id,pass_vk,epoch,expiry,class,sig}` with `PassClass::{Contact, Invite{max_messages:u8}}` (`:212-226`); `Message{ciphertext,sent_time,admission}`; `Admission::{Pass{cert,msg_sig}, Owner{sig}, Quarantine{ghost:Option<GhostProof>}}`; `GhostProof{cert,scoped_payload,ghost_sig}` (`:229-249`).
- `decode` = size check, ciborium decode, re-encode and compare (`:261-273`). Its four refusal reasons: over the byte limit, a count over its bound, not CBOR of this type, non-canonical CBOR.

### `C/src/signed.rs` (225): hand-rolled byte layouts
Domain-prefixed, big-endian, never CBOR (`:1-6`).
- `address_of(vk) = blake3::derive_key("freenet-mail address v1", vk)[..24]` (`:18-27`); `address_check` is not read by the contract (`:29-34`).
- `OWNER_SIG_CONTEXT = b"fmail/v2/owner"` (`:56`); `owner_bytes` = `"fmail/owner/v1" ‖ addr ‖ version ‖ pass_epoch ‖ prev_epoch_until ‖ issuer_vk ‖ quarantine ‖ len‖kem_ek ‖ count‖(id‖sent_time)*` (`:58-81`). `owner_vk` is bound only through the address.
- `time_bytes` (`:84`), `cert_bytes` (`:92`), `revocation_bytes` (`:107`), `usage_bytes` (`:117`), `admit_bytes`/`owner_msg_bytes`/`ghost_admit_bytes` = `domain ‖ recipient ‖ sent_time ‖ blake3(ct)` (`:133-157`).
- `inner_binding` (`"fmail/sealed/v1"`, `:169-183`) is the sender's inner signature layout; the contract never sees it, it lives here only for the vectors.
- `message_id = blake3("fmail/msgid/v1" ‖ recipient ‖ blake3(ct) ‖ sent_time ‖ kind ‖ tag)`, tag = pass id, nothing, or the proof certificate's claimed ghost key, zeros if the certificate does not decode (`:189-225`); kinds 0/1/2 (`:14-16`).

### `C/src/merge.rs` (724): the rules
Constants: `WINDOW_BACK=2 d`, `WINDOW_FORWARD=400 d`, `QUARANTINE_FORWARD=2 d`, `SENT_TIME_GRANULARITY=3600`, `EXPIRY_GRANULARITY=86400`, `CIPHERTEXT_SIZES=[8,16,32 KiB]`, `KEM_EK_LEN=1184`, `MAX_SCOPED_PAYLOAD=1024` (`:56-74`); `WEBAPP_REQUESTOR` = the production webapp id `GeQLXX…` (`:76-81`); `DONATION_ACTION="freenet-donation"` (`:84`).
`Rules::V1`: per_pass 8, per_invite 2, pass_pool 64, owner_pool 8, quarantine 16, per_ghost 2, revoked 4096, usage 1024, tombstones 4096 (`:104-116`).

`merge(rules, params, parts)` (`:458-637`):
1. `join_owner` (`:161-181`): every distinct owner record must pass `verify_owner` or the call is refused; winner is max `(version, blake3(encoding))`. `verify_owner` (`:136-157`): address match, KEM key length, tombstone count, ML-DSA-65 `verify_with_context`.
2. Issuer key must be a valid Ed25519 point or refusal `"issuer key"` (`:464`). `join_time` (`:184-203`): highest `(owner_time, sig)` that verifies under the joined issuer; forged ones are skipped; none gives `"no time attestation"`.
3. Liveness (`:478-493`): `expired = expiry + 2 d <= owner_time`; `oldest_epoch = pass_epoch-1` while `owner_time < prev_epoch_until`, else `pass_epoch`; a revocation is live if unexpired and `epoch >= oldest_epoch` (future epochs count); a usage entry if unexpired and `owner_version <= owner.version`.
4. `join_entry` (`:441-455`) keeps one entry per pass id: revocations by `(epoch, expiry, sig)` (`:223-229`), usage by `(used, expiry, sig)` (`:530`). An entry must be filed under its own id, whole-day, live and issuer-signed (`:517-539`). Dead entries never compete.
5. `live_revoked` is taken before the cap (`:562`), so a revocation the cap cuts still filters this merge; caps keep the smallest ids via `pop_last` (`:570-575`). `revocations_full = len >= cap` (`:579`) refuses all pass mail.
6. `Filter::admit` (`:255-285`): padded size, whole hour, window (quarantine uses the 2 d top), not tombstoned, id derives. Then:
   - Pass (`:287-311`): recipient, epoch in `[oldest_epoch, pass_epoch]`, `sent_time < expiry`, whole-day expiry, set not full, not revoked, issuer signature (memo keyed by the whole certificate, `:315-335`), pass-key signature. Limit: 8, or for an invite `min(max_messages - used, 2)` (`:341-350`).
   - Owner (`:277-280`): issuer signature over `owner_msg_bytes`.
   - Quarantine (`:355-377`): Open admits only a message with no proof; RequireGhost needs a proof within size caps, then `verify_proof` (`:379-426`): requestor `WebApp(WEBAPP_REQUESTOR)`, payload equals `ghost_admit_bytes`, certificate canonical, payload signature before the memo, then chain to `params.ghost_master` with action `freenet-donation` (`:429-437`) and `usd >= min_usd`.
   - One message per id: smallest `(len, bytes)` encoding that passed (`:596-625`).
7. `cap_messages` (`:641-724`): pass pool is round robin, `(k, sent_time, id)` with k the position within the pass, take 64; owner pool is the 8 earliest `(sent_time, id)`; quarantine is ordered by `blake3(salt ‖ id)`, `salt = blake3(encode(time))` (`:628`), take 16, at most 2 per ghost key.

### `C/src/sync.rs` (100)
`Summary{owner,time,revoked,usage: 16-byte digests; revoked_count:u32; messages: bytes}` (`:22-30`); each message entry is 16 bytes of id plus 16 of `blake3(encoding)` (`:17-20,39-50`). `delta` (`:68-100`) sends owner/time if digests differ, the whole revoked set if digest or count differs, the whole usage set if its digest differs, and each message whose entry is not listed; empty bytes if nothing differs.

### `C/src/mutant.rs` (245)
Feature-gated seam: 18 `Mutant` variants (`:21-62`), a thread-local switch (`:64-86`), helpers the rules call.

### Other files
- `C/Cargo.toml`: exact pins (`:22-30`), features `contract`, `freenet-main-contract` (default, empty), `plausible-mutants` (`:40-46`), release profile opt-level 3 with measured timings (`:52-60`).
- `C/.cargo/mutants.toml`: cargo-mutants scope (`:11`), 180 s timeout, test list (`:27-39`).
- `C/coverage-allowlist.toml`: empty. `C/mutants-triage.md`: 6 survivors and 2 timeouts of 245 mutants (`:11-23`).
- `tools/inbox-v2-cost/guest/src/lib.rs`: C ABI `alloc`/`validate`/`update` over `is_valid`/`apply` with `Rules::V1` (`:26-70`). `host/src/main.rs`: wasmtime 47.0.4 runner, `fuel` or `time` mode, fresh instance per case (`:21-103`).
- `docs/metadata-fields.toml`: 40 `[[field]]` rows of path/class/why.

## 3. Interfaces

- **Wire**: canonical CBOR as ciborium emits it: structs are text-keyed maps in declaration order, byte fields are byte strings, enums externally tagged, map keys sorted bytes (`C/src/types.rs:1-8`; `C/tests/fixtures/vectors.py:118-119`).
- **stdlib**: `freenet-stdlib =0.12.1`; `#[contract]` only with feature `contract` (`C/src/lib.rs:137`); `ContractInstanceId` decodes the proof's requestor (`C/src/merge.rs:125-129`).
- **Crypto deps**: `ml-dsa =0.1.1`, `ed25519-dalek =2.2.0` (`verify_strict`, `C/src/merge.rs:131-134`), `blake3 =1.8.7`, `ghostkey-verify =0.1.0`, which locks `rsa 0.10.0-rc.19` (`C/Cargo.lock:995-996`).
- **Host imports**: the candidate check greps the WASM for the clock and randomness imports and fails on either (`scripts/check-inbox-candidate.sh:31-39`).
- **Env**: `FORK_SEEDS`, `FORK_SEED` (`C/tests/merge_laws.rs:15`), `INBOX_CAPTURE_DIR` (`:772`), `INBOX_V2_DUMP_DIR` (`C/tests/worst_case.rs:202`), `VECTORS_PYTHON`, `INBOX_COST_BUDGET`, `COVERAGE_NIGHTLY`, `FDEV`.
- **Who reads this crate**:
  - `protocol/src/inbox_v2.rs:410` and `delegates/mail-delegate/src/tests/inbox_vectors.rs:13` include `vectors.txt`.
  - Delegate tests call `apply`/`is_valid` (`delegates/mail-delegate/src/tests/inbox_v2.rs:7-8`), include `ghost-keys.txt` (`tests/inbox_jobs.rs:995`) and the text of `C/Cargo.toml` (`:1603-1613`).
  - `delegates/mail-delegate/src/inbox_jobs.rs:83-85` embeds `inbox_v2_code_hash` in non-test code.
  - `ui/src/v2/mod.rs:27-29` embeds candidate bytes and hash; `ui/src/ghostkeys.rs:768` the Ghost Key fixture.
  - `protocol/src/inbox_v2.rs:33-51` duplicates the windows and caps by hand.
- **Who can write what**:
  - ML-DSA owner key: the owner record (mode, epoch, grace, KEM key, issuer key, tombstones).
  - Issuer Ed25519 key named in that record: time, pass certificates, revocations, usage entries, owner mail.
  - A pass key: mail under its certificate.
  - A Ghost Key donor: quarantine mail in RequireGhost.
  - Anyone: unsigned quarantine mail in Open, and redelivery of any genuine piece.

## 4. Invariants and where they are held

| Rule | Code | Test |
|---|---|---|
| Valid = fixed point of merge | `C/src/lib.rs:70-78` | every `is_valid` assertion; `C/tests/merge_laws.rs:558-596` |
| One accepted encoding | `C/src/types.rs:261-273` | `C/tests/canonical.rs:46-69`; guard `scripts/check-guards-fail-inbox.sh:12-15` |
| Counts and bytes refused before verification, summed per call | `C/src/types.rs:43-101`, `C/src/lib.rs:116-120` | `C/tests/bounds.rs:127-349` |
| Forged or failing owner refuses the call | `C/src/merge.rs:161-181` | `C/tests/tamper.rs:350-449` |
| Everything else failing is dropped silently | `C/src/merge.rs:23-29` | `C/tests/tamper.rs:207-321`, `C/tests/ghost.rs` |
| A pruned revocation never filters; dead entries never win a join | `C/src/merge.rs:467-469,562` | `C/tests/rules.rs:407-421`; `C/tests/merge_laws.rs:899-973` |
| Current epoch's revocation beats the previous one's | `C/src/merge.rs:223-229` | `C/tests/rules.rs:1084-1123` |
| Full revocation set admits no pass mail | `C/src/merge.rs:579` | `C/tests/rules.rs:653-810` |
| A usage entry counts only once its owner version is joined | `C/src/merge.rs:493` | `C/tests/rules.rs:493-541` |
| Open refuses any proof; RequireGhost requires one | `C/src/merge.rs:355-364` | `C/tests/ghost.rs:304-324,67-72` |
| Message id binds kind and tag | `C/src/signed.rs:217-225` | `C/tests/ghost.rs:405-483` |
| Owner signature under its own context | `C/src/merge.rs:154` | `C/tests/tamper.rs:413-432`; `C/tests/vectors.rs:167-184` |
| No related contracts; exact params shape | `C/src/lib.rs:44,103` | `C/tests/rules.rs:926-1044` |
| Largest valid state fits 4 MiB; summary under a quarter of the state | `C/src/types.rs:32-35` | `C/tests/worst_case.rs:66-89,260-333` |
| Byte layouts fixed | `C/src/signed.rs` | `C/tests/vectors.rs`, `scripts/check-inbox-vectors.sh` |
| Seam never in a contract build | `C/src/lib.rs:19-20` | compile-time |

**Error versus drop.** Refused: undecodable or non-canonical part; count or byte bound; call sums; any failing owner record; invalid issuer point; no verifying time; unsupported `UpdateData`. Dropped without error: lower or forged times beside a good one, bad or dead revocations and usage entries, any message that fails the filter or loses a cap. Senders must read back (`C/src/merge.rs:23-29`).

**Time model.** `owner_time` is whatever the issuer key signed; the join takes the maximum and the contract puts no bound on a jump. `sent_time` must be a whole hour inside `[owner_time-2 d, owner_time+400 d]`, or `+2 d` for quarantine. Expiries are whole days. Epochs: `pass_epoch`, with `pass_epoch-1` admitted while `owner_time < prev_epoch_until` (the delegate uses 14 days, `delegates/mail-delegate/src/inbox_v2.rs:59-60`; 0 = emergency).

## 5. State of things

- Frozen: nothing of v2. Candidate: `published-contract/candidate/inbox_v2` (1,030,424 bytes: a 40-byte header then WASM) and `inbox_v2_code_hash` = `GpuWR879xBi13sSeDhoT1zNzEhVmMBFSEr1aoJGCc8a6`.
- The UI already embeds it (`ui/src/v2/mod.rs:27`); `scripts/check-release-delegate.sh:72-75` refuses a release while the UI references `candidate/`.
- Not done: the freeze into `published-contract/wasm/v2/` with a `frozen.sha256` row (`published-contract/candidate/README.md:8-9`); the README section the disclosure gate claims to feed (section 8); the `release.sh` hook ("comes with C3", `AGENTS.md:293`).

## 6. Tests

Run from `C/`: `cargo test --locked`, and again with `--features plausible-mutants` (`scripts/check-excluded-crates.sh:28-53` does both; `cargo make test` reaches it through `test-excluded`, `Makefile.toml:921-924`). `cargo make test-inbox` is the v1 contract, not this one (`Makefile.toml:926-933`). Needs rustc 1.99.1 (`rust-toolchain.toml:12`) and crates.io access; no node.

- `tests/common/mod.rs`: fixed keys (ML-DSA seed `[7;32]`, issuer `[9;32]`), builders for every signed piece, `non_canonical_forms` (`:298-357`).
- `tests/merge_laws.rs`: seeded fuzzer, 48 seeds by default, 6 triples per seed, under `TINY` caps and V1; an at-cap universe every 20th seed (`:31-47,660-708`). Checks validity, idempotence, commutativity, absorption, associativity-or-heal within 4 passes (`:528-596`), delta path equals whole-state path (`:600-644`), non-canonical forms refused. Every merge goes through `oracle::check` (`:472-518`). It asserts both law branches ran and every V1 cap overflowed (`:828-857`). `capture_states_for_fdev` writes the fdev corpus (`:770-788`). With the feature, two mutants must fail through the oracle (`:864-897`).
- `tests/oracle/mod.rs`: independent model using only `signed.rs`, the types and `ghostkey-verify`; own literals in `SPEC_V1` (`:59-75`), pinned by `the_oracles_numbers_are_the_crates` (`merge_laws.rs:979-1001`). It does not model one pass id with two limits (`:384-388`).
- `tests/rules.rs` (41 tests): each filter, cap and join at its boundary, plus the contract trait and the summary.
- `tests/tamper.rs` (19): one field changed per signed item with everything else consistent.
- `tests/ghost.rs` (17): proof checks, tiers, per-ghost cap, memo, Open-versus-RequireGhost.
- `tests/bounds.rs` (7), `tests/canonical.rs` (3).
- `tests/worst_case.rs` (4): largest and widest states, and `dump_the_cost_gate_inputs` (`:200-250`).
- `tests/vectors.rs` (5): 35 of the 36 lines of `vectors.txt`; `mldsa_prefix_sealed` is checked by the delegate, not here.
- `tests/metadata_disclosure.rs` (4): walks populated states in both modes and requires path-set equality with the toml (`:188-201`).
- `tests/plausible_mutants.rs`: 18 generated tests; each scenario returns no broken law on real rules and exactly `fingerprint(m)` under the mutant (`:564-611`).

Scripts: `scripts/check-inbox-vectors.sh` (cbor2 5.8.0, b3sum 1.8.5); `scripts/check-inbox-candidate.sh` (fdev, linux/amd64, `flock`); `scripts/check-inbox-cost.sh` (budget 37e9 fuel, calibration `:17-33`, exact return values `:65-72`); `scripts/check-fdev-verify-merge.sh` (fdev v0.2.151, 200 cases, at least 12 states); `scripts/check-branch-coverage.sh` (nightly-2025-08-08, cargo-llvm-cov 0.6.21, `FORK_SEEDS=3`); cargo-mutants 27.1.0 in 8 shards with `FORK_SEEDS=12`, then `scripts/check-mutants-triage.sh` (`.github/workflows/inbox-v2-gates.yml:33-36,71-141`). Nightly 1000 seeds: `.github/workflows/inbox-v2-laws.yml:23-27`. Guards alone: `GUARD_ONLY='^inbox v2: ' scripts/check-guards-fail.sh --no-wasm` (`scripts/check-guards-fail-inbox.sh:3`). The four slow gates are not required checks (`inbox-v2-gates.yml:5-8`).

## 7. Changing it safely

- **Any edit under `C/src`, `C/Cargo.toml` or `C/Cargo.lock`** changes the candidate. Rebuild with `scripts/build-wasm.sh contracts/inbox-v2 contract contract …` and rewrite the hash (`published-contract/candidate/README.md:38-46`); CI is the canonical builder and uploads `inbox-v2-candidate-<sha>`.
- **Trap: the delegate embeds the code hash** (`delegates/mail-delegate/src/inbox_jobs.rs:83-85`), so a contract change also changes the delegate's bytes and its candidate must be rebuilt too. Every test-network inbox id moves with the hash.
- **Layout or field change**: update `vectors.py` and regenerate `vectors.txt` deliberately, the copy in `protocol/src/inbox_v2.rs`, `docs/metadata-fields.toml`, and the exact merged sizes in `scripts/check-inbox-cost.sh:65-72`.
- **Number change**: `Rules::V1`, `types::MAX_*`, `oracle::SPEC_V1`, the protocol constants (`protocol/src/inbox_v2.rs:33-51`; I found no test pinning them to the contract), the delegate's bump threshold (`delegates/mail-delegate/src/inbox_v2.rs:62`), and the cost calibration.
- **Rule change**: the oracle must change with it; a plausible new mistake needs a `Mutant`, a scenario and a fingerprint; new branch arms need a test or an allowlist row; a new test file must be listed in `C/.cargo/mutants.toml` (`inbox-v2-gates.yml:91-98`).
- **Textual traps**: the guards patch exact source lines (`scripts/check-guards-fail-inbox.sh:12-41`), so reformatting those lines breaks them; triage lines are keyed by function name (`scripts/check-mutants-triage.sh:80-81`), so a rename stales them; a delegate test matches the literal `ed25519-dalek = { version = "=2.2.0"` in `C/Cargo.toml`.
- **After the freeze** the rules cannot be patched: a fix is a new generation plus a migration (`published-contract/README.md:78-100`). No v2 migration code exists.

## 8. Docs versus code

1. `published-contract/candidate/README.md:15` says of `inbox_v2` "Nothing embeds it yet". The UI embeds the bytes (`ui/src/v2/mod.rs:27`) and the delegate the hash (`inbox_jobs.rs:84`).
2. `docs/metadata-fields.toml:2`, `C/tests/metadata_disclosure.rs:7-8` and `AGENTS.md:292-293` speak of the README's "What others can see". `README.md` has no such section and nothing generates one; the gate is toml-versus-state only.
3. `AGENTS.md:288-289` and `C/mutants-triage.md:86-87` say the rules hold only one-line calls into the seam. Mutant logic is inline at `C/src/merge.rs:224-227`, `:316-326` (a copy of the certificate check), `:471-477`, `:564-568`, `:602-606`, `:612-617` and `C/src/sync.rs:43-46`.
4. `C/src/merge.rs:23-25` lists two refusals inside `merge`; the invalid issuer point is a third (`:464`). `AGENTS.md:277-283` omits both the time and the issuer refusal.
5. `AGENTS.md:258-260` says the numbers are in `merge::Rules`. The windows, granularities, sizes and decode bounds are module constants that tests cannot shrink.
6. `C/tests/bounds.rs:6-7` says a summary is held to the counts. `Summary.messages` is an unbounded byte string, limited only by `MAX_INPUT_BYTES` (`C/src/sync.rs:22-30`).
7. `C/tests/tamper.rs:517-520` says a changed `issuer_vk` "makes every later merge refuse". The test sends no time under the new key; see 9.2.
8. `C/mutants-triage.md:33-36` says `pass_id` is the one field outside the usage rank; `owner_version` is outside it too (`C/src/merge.rs:530`). The conclusion still holds because the signature covers it.
9. `C/Cargo.toml:16` and `scripts/build-wasm.sh:12` say the build is `--locked`; the fdev invocation passes no such flag (`scripts/build-wasm.sh:99-102`).
10. `C/tests/oracle/mod.rs:23` cites decision 23 for the usage rank; `C/src/merge.rs:247` cites it for the bump at 2048.

## 9. Suspected weak spots

1. **Quarantine rank is grindable** (mechanism certain; whether accepted by design unknown). The salt is public state and the id depends on the ciphertext (`C/src/merge.rs:628,662-665`), so a sender can try ciphertexts until its rank beats those held and evict honest strangers until `owner_time` moves. `C/tests/plausible_mutants.rs:163-164` only rules out choosing by `sent_time`. In Open mode such mail is unsigned.
2. **Issuer rotation is possible and rewinds time** (medium-high; not run). If one call carries a higher-version owner record naming a new issuer and a time signed by it, `join_time` ignores every older time, so `owner_time` can go down and all old issuer-signed items drop. Owner-only.
3. **A join discards the lower-ranked live entry for good** (medium). If a higher-epoch revocation, or a higher-`used` usage entry, has the earlier expiry, it wins, later expires, and the entry it displaced is gone unless redelivered. The fuzzer builds such crossing pairs (`C/tests/merge_laws.rs:275-281,318-320`) but the oracle judges one merge at a time. Safe only while the client signs the largest expiry (`delegates/mail-delegate/src/inbox_v2.rs:308-314`).
4. **Same-version owner fork** (certain): the digest breaks the tie and the loser's tombstones vanish, so its drained mail can return inside the window. Decision 25 on the delegate only detects it.
5. **`WEBAPP_REQUESTOR` is compiled in** (certain): a move to another webapp contract would break RequireGhost admission for every v2 inbox until a new generation.
6. **No bound on an `owner_time` jump** (certain): one far-future attestation is permanent, except through 2.
7. **`rsa 0.10.0-rc.19`**, a release candidate, sits inside rules that cannot be patched after the freeze (fact; risk is a judgement).
8. **Usage cap** (documented, `C/src/merge.rs:337-340`): past 1024 live entries an invite with a large id reads as unused.
9. **Triage by name** (low): the line `replace > with >= in Filter<'_>::admit_quarantine` would also excuse the same mutation of the payload-size comparison on the next line if its test went away.

## 10. Open work and documented limits

- Freeze at v0.3.0 (`C/Cargo.toml:17`, `.release-freeze`).
- README disclosure section and the `release.sh` refusal ("C3").
- Full revocation set: a routine bump does not escape it; the client is expected never to get there (`C/src/merge.rs:242-248`).
- Two cargo-mutants timeouts are left as hangs because a fail-fast rewrite would change the candidate (`C/mutants-triage.md:66-74`).
- Cost calibration must be re-measured when the work mix moves (`scripts/check-inbox-cost.sh:31-33`).
- No `TODO`/`FIXME` markers in the owned paths.

## 11. Decisions cited, and questions

No file in the tree defines the numbered decisions, the "plan" (`plan 5.3`, `§5.5`, "surgery plan"), the review rounds (`r1`…`r4`) or `C2`/`C3`. Meanings as the citations give them:

- 1: what a Ghost Key proof signs, and the pinned requestor (`protocol/src/inbox_v2.rs:287`).
- 2 (with 19): a proof is attached only for RequireGhost recipients; caps (`ui/src/v2/send.rs:182-184`).
- 3: one message per id, smallest encoding. 4: owner mail is signed by the issuer key.
- 5: time join; caps keep the smallest ids. 6: the issuer key never changes.
- 8: a worst call stays under 2.5 s. 9: drain before signing a change that drops mail; senders read back and reseal.
- 10: only revocations left after pruning filter; a full set refuses passes.
- 11: the id binds the pass id or ghost key. 12: per-part count bounds at decode. 13: 16+16-byte summary entries.
- 14: 2-day quarantine lead, 16 slots, 2 per ghost. 16: bounds summed over a call. 17: whole-day expiries.
- 18: previous-epoch grace. 19: Open refuses proofs. 20: map keys are disclosure paths.
- 21: delegate keeps the newest 1024 strangers. 22: revocation rank is epoch first.
- 23: bump at 2048 revocations; revocations sign the largest expiry; `used` only grows.
- 24: ML-DSA context strings. 25: delegate notes a foreign owner record.
- 7 and 15 are never cited anywhere.

Questions I could not answer from the tree:
- Does `fdev build` apply `--locked`?
- Does core ever send `StateAndDelta` whose parts together exceed a bound (refused by `C/src/lib.rs:116-120`), call `get_state_delta` with an empty summary (it would return `Deser`), or cap state size below 4.25 MiB?
- Where does the "summary under a quarter of the state" requirement come from?
- Is the committed candidate the current source's build (not checked: no build allowed)?
- Does `nightly-2025-08-08` still build the locked dependencies?
- What happens to test-network inboxes each time the candidate hash changes?

---

## Appendix: the reader's structured summary

contracts/inbox-v2 is the next-generation inbox contract: one instance per identity, parameters `0x02 ‖ addr24 ‖ ghost_master32`. Its state is canonical CBOR holding the ML-DSA-signed owner record, an issuer-signed clock, revocations, invite usage entries and up to 88 undrained ciphertexts. All rules live in `merge::merge`: `update_state` merges the held state with the caller's parts, and `validate_state` accepts a state only if merging it alone returns it. Admission is by pass (issuer-certified Ed25519 key), by owner (issuer signature), or through quarantine (unsigned in Open mode, a Ghost Key donation proof in RequireGhost mode). Only undecodable input, count/byte bounds, a failing owner record, an invalid issuer key or no verifying time refuse a call; everything else is dropped silently.

State: a candidate, not frozen. The WASM is committed under published-contract/candidate/ and already embedded by the UI; its code hash is embedded by the mail delegate. `.release-freeze` blocks releases until v0.3.0. Nothing was built or run for this map; it is from reading only.

Testing is heavy: a seeded merge-law fuzzer held to an independent oracle, 18 plausible-mutant scenarios, byte vectors from a Python second implementation, tamper/bounds/worst-case suites, a metadata disclosure gate, and four slow non-required gates (branch coverage, cargo-mutants, fdev verify-merge, wasmtime fuel cost).

The numbered design decisions and the "plan" the code cites are not defined anywhere in the tree. The main risks I found are a grindable quarantine rank, issuer rotation rewinding time, and several stale doc claims.

### Key facts

- One rule set: `is_valid` decodes, merges the state alone and requires equality; `apply` merges held state plus the caller's parts; production always uses `Rules::V1` (contracts/inbox-v2/src/lib.rs:70-108,146,160).
- Parameters are exactly 57 bytes, `0x02 ‖ addr24 ‖ ghost_master32`; the address is blake3 derive_key("freenet-mail address v1", ML-DSA vk)[..24], so the contract id is per inbox (contracts/inbox-v2/src/lib.rs:30-52; contracts/inbox-v2/src/signed.rs:18-27).
- Caps in `Rules::V1`: per_pass 8, per_invite 2, pass_pool 64, owner_pool 8, quarantine 16, per_ghost 2, revoked 4096, usage 1024, tombstones 4096 (contracts/inbox-v2/src/merge.rs:104-116).
- Windows and sizes are module constants, not in `Rules`: back 2 d, forward 400 d, quarantine forward 2 d, sent_time whole hours, expiries whole days, ciphertext 8/16/32 KiB, KEM key 1184 bytes, scoped payload at most 1024 bytes (contracts/inbox-v2/src/merge.rs:56-74).
- Decode bounds are separate constants (88 messages, 4096 revoked, 1024 usage, 4096 tombstones, 4 MiB + 256 KiB) enforced entry by entry and summed over the caller's parts; a test pins them to Rules::V1 (contracts/inbox-v2/src/types.rs:28-66; contracts/inbox-v2/tests/bounds.rs:353-360).
- Canonical CBOR is enforced by re-encoding and comparing; an `InboxDelta` must spell all five keys (contracts/inbox-v2/src/types.rs:261-273,134-143).
- Refusals inside merge: any distinct owner record failing verification, an invalid issuer Ed25519 point, or no time verifying under the joined issuer; `apply` also refuses any UpdateData other than State/Delta/StateAndDelta (contracts/inbox-v2/src/merge.rs:161-203,464; contracts/inbox-v2/src/lib.rs:94-104).
- Key roles: the ML-DSA owner key signs only the owner record (context `fmail/v2/owner`); the Ed25519 issuer key named in it signs time, pass certificates, revocations, usage entries and owner mail; pass keys sign pass mail; Open-mode quarantine mail carries no signature (contracts/inbox-v2/src/signed.rs:56-157; contracts/inbox-v2/src/merge.rs:277-311,361-363).
- Epoch grace: passes of `pass_epoch-1` are admitted and their revocations kept while `owner_time < prev_epoch_until`; revocations rank by (epoch, expiry, sig), usage by (used, expiry, sig), and dead entries never compete in a join (contracts/inbox-v2/src/merge.rs:223-229,478-493,517-539).
- A revocation set holding exactly its cap refuses all pass mail; `live_revoked` is taken before the cap so cut revocations still filter that merge (contracts/inbox-v2/src/merge.rs:562-579).
- RequireGhost proofs must name the hard-coded production webapp id as requestor, sign `fmail/ghost-admit/v1 ‖ recipient ‖ sent_time ‖ blake3(ct)`, and chain to `params.ghost_master` as a `freenet-donation` of at least `min_usd`; Open refuses any message carrying a proof (contracts/inbox-v2/src/merge.rs:76-84,355-437).
- Message caps: pass pool round robin by (k, sent_time, id); owner pool the 8 earliest; quarantine ordered by blake3(salt ‖ id) where salt = blake3(encoded time attestation), at most 2 per ghost key (contracts/inbox-v2/src/merge.rs:628,641-724).
- Summary entries are 16 bytes of id plus 16 of encoding digest; the delta resends the whole revoked or usage set when its digest (or the revoked count) differs (contracts/inbox-v2/src/sync.rs:17-30,68-100).
- The candidate is embedded already: the UI includes the WASM and hash, and the delegate's non-test code includes the hash, so any contract change also changes the delegate's bytes (ui/src/v2/mod.rs:27-29; delegates/mail-delegate/src/inbox_jobs.rs:83-85).
- Tests are run with `cargo test --locked` in contracts/inbox-v2, then again with `--features plausible-mutants`; `cargo make test-inbox` is the v1 contract (scripts/check-excluded-crates.sh:28-53; Makefile.toml:921-933).
- The fuzzer runs 48 seeds by default (FORK_SEEDS / FORK_SEED), holds every merge to tests/oracle, and asserts both law branches and every V1 cap overflow were reached (contracts/inbox-v2/tests/merge_laws.rs:15,31-47,472-518,828-857).
- The cost gate pins the exact merged-state sizes its cases must return (1670880, 3791380, 3791383) and a 37e9 fuel budget, so any encoding or worst-case builder change needs those numbers updated (scripts/check-inbox-cost.sh:17-33,42,65-72).
- Guards patch exact source lines and triage lines are keyed by function name, so reformatting or renaming in merge.rs/types.rs can break them without a behaviour change (scripts/check-guards-fail-inbox.sh:12-41; scripts/check-mutants-triage.sh:80-81).
- A delegate test matches the literal text `ed25519-dalek = { version = "=2.2.0"` in the contract's Cargo.toml (delegates/mail-delegate/src/tests/inbox_jobs.rs:1603-1613).
- The four slow gates (branch coverage, cargo-mutants, fdev verify-merge, cost) run only on PRs touching the contract and nightly, and are not required checks (.github/workflows/inbox-v2-gates.yml:5-8,13-27).

### Docs or comments that disagree with the code

- **[medium]** Claim (published-contract/candidate/README.md:15): The inbox_v2 candidate: "Nothing embeds it yet." **Reality:** The UI includes the candidate WASM and its code hash, and the mail delegate includes the code hash in non-test code. Evidence: ui/src/v2/mod.rs:27-29; delegates/mail-delegate/src/inbox_jobs.rs:83-85
- **[medium]** Claim (AGENTS.md:292-293; docs/metadata-fields.toml:2; contracts/inbox-v2/tests/metadata_disclosure.rs:7-8): docs/metadata-fields.toml is the source of the README's "What others can see", so a field cannot become public without the README line. **Reality:** README.md has no such section and no reference to the toml; nothing generates one. The gate only compares the toml with a populated state. Evidence: README.md:1-105 (headings at 1,17,47,82,92,101); contracts/inbox-v2/tests/metadata_disclosure.rs:188-201
- **[low]** Claim (AGENTS.md:288-289; contracts/inbox-v2/mutants-triage.md:86-87): The mutated logic lives in src/mutant.rs; the rules hold one-line calls and no copy of a rule. **Reality:** Several mutants are implemented inline in the rules under cfg, including a ten-line copy of the certificate check for CertMemoByPassId. Evidence: contracts/inbox-v2/src/merge.rs:224-227,316-326,471-477,564-568,602-606,612-617; contracts/inbox-v2/src/sync.rs:43-46
- **[low]** Claim (contracts/inbox-v2/src/merge.rs:23-25 (module doc); AGENTS.md:277-283 lists even fewer): Within merge, only an owner record that fails, or no time that verifies, refuses the input. **Reality:** An issuer_vk that is not a valid Ed25519 point is a third refusal ("issuer key"). AGENTS.md's list omits both the time refusal and this one, and the refusal of unsupported UpdateData variants. Evidence: contracts/inbox-v2/src/merge.rs:464; contracts/inbox-v2/src/lib.rs:102-103
- **[low]** Claim (AGENTS.md:258-260): The contract's numbers are in merge::Rules, passed as an argument so tests can run with small caps. **Reality:** Only the nine caps are in Rules. Windows, granularities, ciphertext sizes, KEM length, scoped-payload cap and the decode bounds are constants that tests cannot shrink; the decode bounds stay at V1's values under any Rules. Evidence: contracts/inbox-v2/src/merge.rs:56-74; contracts/inbox-v2/src/types.rs:28-35
- **[low]** Claim (contracts/inbox-v2/tests/bounds.rs:6-7 (module doc)): Every part a contract call decodes (a state, a delta, a summary) is held to the count bounds as it is decoded. **Reality:** A summary's message list is a plain byte string with no entry bound; only MAX_INPUT_BYTES limits it. Evidence: contracts/inbox-v2/src/sync.rs:22-30,68-74
- **[medium]** Claim (contracts/inbox-v2/tests/tamper.rs:517-520 (test comment)): An owner record that changes issuer_vk makes every later merge refuse, because no time attestation verifies under the new key. **Reality:** The test sends the new owner record without a time. By the code, a call that also carries a time signed by the new issuer would merge: join_time considers only times that verify under the joined issuer. Not run. Evidence: contracts/inbox-v2/src/merge.rs:184-203; contracts/inbox-v2/tests/tamper.rs:522-537
- **[low]** Claim (contracts/inbox-v2/mutants-triage.md:33-36): For the join_entry survivor: the one field left out of the rank is pass_id. **Reality:** The usage rank (used, expiry, sig) also leaves out owner_version. The equivalence argument still holds because the signature covers owner_version, but the stated reason is incomplete. Evidence: contracts/inbox-v2/src/merge.rs:530; contracts/inbox-v2/src/types.rs:200-209
- **[low]** Claim (contracts/inbox-v2/Cargo.toml:16; scripts/build-wasm.sh:12; delegates/mail-delegate/src/tests/inbox_jobs.rs:1597-1598): The crate is built only by scripts/build-wasm.sh with --locked. **Reality:** The script's fdev invocation passes no --locked flag; whether fdev applies it cannot be told from the tree. Evidence: scripts/build-wasm.sh:99-102
- **[low]** Claim (contracts/inbox-v2/tests/oracle/mod.rs:23): Decision 23 is the usage rank (used, expiry, sig). **Reality:** Elsewhere decision 23 is the client bumping the epoch at 2048 revocations and signing the largest expiry. The citations are inconsistent and no definition exists in the tree. Evidence: contracts/inbox-v2/src/merge.rs:247; delegates/mail-delegate/src/inbox_v2.rs:61-62,308-314

### Suspected issues (leads from reading, not confirmed)

- **Quarantine eviction can be ground by a sender** (confidence: high). Quarantine keeps the 16 lowest blake3(salt ‖ id). The salt is the hash of the public time attestation and the id depends on the ciphertext, so a sender can try ciphertexts until its rank beats those held, evicting honest strangers until owner_time next moves. In Open mode such mail is unsigned, and anyone can also restamp another sender's ciphertext under a new sent_time. The mutant test only rules out choosing a place by sent_time. Mechanism is certain from the code; whether it is accepted by design is not stated. Evidence: contracts/inbox-v2/src/merge.rs:628,662-665,704-720; contracts/inbox-v2/tests/plausible_mutants.rs:163-164
- **Issuer rotation is possible and can move owner_time backwards** (confidence: medium). If one call carries a higher-version owner record naming a new issuer_vk together with a time signed by that key, join_time ignores all older attestations (they no longer verify), so the new, possibly lower, time wins and every item signed by the old issuer is dropped. Only the owner can do this, but it contradicts the stated assumption that the issuer never changes and that time only advances. From reading; not run. Evidence: contracts/inbox-v2/src/merge.rs:184-203,463-465; contracts/inbox-v2/tests/tamper.rs:517-537
- **A join discards the lower-ranked live entry for good** (confidence: medium). join_entry keeps one revocation or usage entry per pass id. If the winner (higher epoch, or higher `used`) has the earlier expiry, it later expires and the displaced entry is gone unless someone redelivers it: a revoked pass or a used invite can come back. The fuzzer builds such crossing pairs, but the oracle judges one merge at a time. Safe only while the client always signs the largest expiry, which the delegate does today; the contract does not enforce it. Evidence: contracts/inbox-v2/src/merge.rs:441-455,517-539; contracts/inbox-v2/tests/merge_laws.rs:275-281,318-320; delegates/mail-delegate/src/inbox_v2.rs:308-314
- **Two owner records at one version: the loser's tombstones are lost** (confidence: high). The owner join is max (version, digest). Two devices signing the same version resolve by digest, and the losing record's tombstones vanish, so mail only it had drained can return while inside the window. Decision 25 on the delegate detects another node but the contract offers no merge of tombstones. Evidence: contracts/inbox-v2/src/merge.rs:161-181,264
- **Webapp contract id is compiled into the rules** (confidence: high). WEBAPP_REQUESTOR is the production webapp id. AGENTS.md says the container contract exists so the app could move to a new webapp contract; such a move would make every RequireGhost proof fail in every v2 inbox until a new inbox generation and migration. Evidence: contracts/inbox-v2/src/merge.rs:76-81,399
- **No bound on an owner_time jump** (confidence: high). The time join takes the maximum issuer-signed owner_time with no step limit. One far-future attestation (client bug or issuer key misuse) permanently moves the window and prunes every revocation, usage entry and pass; the only step guard is in the delegate. Evidence: contracts/inbox-v2/src/merge.rs:184-203,478
- **A release-candidate RSA crate inside unpatchable rules** (confidence: medium). ghostkey-verify 0.1.0 locks rsa 0.10.0-rc.19, which performs the notary signature check for RequireGhost admission. After the freeze the contract cannot be patched. This is a fact about the lockfile; how much risk it carries is a judgement. Evidence: contracts/inbox-v2/Cargo.lock:995-996; contracts/inbox-v2/Cargo.toml:27
- **One pass id with two limits is not modelled** (confidence: low). If the issuer signs a Contact and an Invite certificate (or invites with different max_messages) under one pass id, cap_messages applies each message's own limit against a shared counter. The oracle returns an error rather than model it and the fuzzer never builds it, so this path is untested. Evidence: contracts/inbox-v2/src/merge.rs:646-687; contracts/inbox-v2/tests/oracle/mod.rs:383-388
- **Triage by mutant name can excuse a second mutant** (confidence: medium). The triage line `replace > with >= in Filter<'_>::admit_quarantine` is written for the certificate-size comparison but, with positions stripped, would also excuse the same mutation of the scoped-payload comparison on the next line if its test were ever removed. Evidence: contracts/inbox-v2/mutants-triage.md:47-59; contracts/inbox-v2/src/merge.rs:365-366; scripts/check-mutants-triage.sh:14-15,80-81

### Open work

- Freeze the candidate into published-contract/wasm/v2/ with a frozen.sha256 row at v0.3.0; until then every source change moves the bytes (contracts/inbox-v2/Cargo.toml:14-17; published-contract/candidate/README.md:8-9; .release-freeze:1-11).
- A release is refused while the UI references published-contract/candidate/, which it does for inbox v2 (scripts/check-release-delegate.sh:72-75; ui/src/v2/mod.rs:27).
- The README "What others can see" section and the release.sh refusal tied to it do not exist yet ("comes with C3", AGENTS.md:292-293).
- No migration path from v1 inboxes to v2 exists in the owned paths; a post-freeze rule fix is a new generation plus migration (published-contract/README.md:78-100).
- Documented limit: past 1024 live usage entries an invite with a large pass id reads as unused (contracts/inbox-v2/src/merge.rs:337-340).
- Documented limit: a full revocation set refuses every pass and a routine bump does not escape it; the client is expected to bump at 2048 (contracts/inbox-v2/src/merge.rs:242-248).
- Two cargo-mutants timeouts are left as hangs because a fail-fast rewrite would change the candidate's bytes (contracts/inbox-v2/mutants-triage.md:66-74).
- The cost calibration must be re-measured whenever a change moves the mix of work (scripts/check-inbox-cost.sh:28-33).
- The caller-side check that `inner_binding` is given the body hash, not the ciphertext hash, is deferred to where the envelope is built ("C2", contracts/inbox-v2/src/signed.rs:165-168).
- coverage-allowlist.toml is empty by intent; any new unvisited branch arm needs a test or a reasoned entry (contracts/inbox-v2/coverage-allowlist.toml:1-9).
- The slow gates are not required checks, so a red run does not block a merge (.github/workflows/inbox-v2-gates.yml:5-8; .github/workflows/inbox-v2-laws.yml:3-5).

### Open questions

- Where are decisions 1-25, the "plan" (plan 5.3/5.4/5.5, "surgery plan §5.x"), the review rounds r1-r4 and items C2/C3 defined? No file in the tree defines them; decisions 7 and 15 are never cited at all. The map lists what each cited decision appears to mean.
- Does `fdev build` apply --locked? scripts/build-wasm.sh passes no such flag although three places say the build is locked.
- Does freenet-core ever send StateAndDelta (or several UpdateData entries) whose parts together exceed a count bound? Such a call is refused whole (contracts/inbox-v2/src/lib.rs:116-120), e.g. a full valid state plus a one-message delta.
- Does core ever call get_state_delta with an empty summary? It would return ContractError::Deser (contracts/inbox-v2/src/lib.rs:180-181).
- Does core cap contract state size below the 4 MiB + 256 KiB the contract accepts?
- Where does the requirement that a summary be under a quarter of the state come from (contracts/inbox-v2/tests/merge_laws.rs:608; contracts/inbox-v2/tests/worst_case.rs:87)?
- Is the committed candidate actually the current source's build, and do the tests pass at this commit? Nothing was built or run.
- Is the grindable quarantine rank an accepted property of Open mode, or an oversight?
- Does the pinned coverage toolchain nightly-2025-08-08 still build the locked dependencies, given the stable pin is 1.99.1?
- What happens to test-network inboxes each time the candidate's code hash changes: is their data simply abandoned under the old contract id?
- No test was found pinning the client's duplicated caps and windows (protocol/src/inbox_v2.rs:33-51) to the contract's constants; is there one elsewhere, or do they rely only on the interop tests?

### Coverage

Read in full: `contracts/inbox-v2/src/lib.rs`, `contracts/inbox-v2/src/types.rs`, `contracts/inbox-v2/src/signed.rs`, `contracts/inbox-v2/src/sync.rs`, `contracts/inbox-v2/src/merge.rs`, `contracts/inbox-v2/src/mutant.rs`, `contracts/inbox-v2/Cargo.toml`, `contracts/inbox-v2/freenet.toml`, `contracts/inbox-v2/.cargo/mutants.toml`, `contracts/inbox-v2/coverage-allowlist.toml`, `contracts/inbox-v2/mutants-triage.md`, `contracts/inbox-v2/tests/common/mod.rs`, `contracts/inbox-v2/tests/oracle/mod.rs`, `contracts/inbox-v2/tests/merge_laws.rs`, `contracts/inbox-v2/tests/rules.rs`, `contracts/inbox-v2/tests/tamper.rs`, `contracts/inbox-v2/tests/ghost.rs`, `contracts/inbox-v2/tests/bounds.rs`, `contracts/inbox-v2/tests/canonical.rs`, `contracts/inbox-v2/tests/worst_case.rs`, `contracts/inbox-v2/tests/vectors.rs`, `contracts/inbox-v2/tests/metadata_disclosure.rs`, `contracts/inbox-v2/tests/plausible_mutants.rs`, `contracts/inbox-v2/tests/fixtures/vectors.py`, `contracts/inbox-v2/tests/fixtures/README.md`, `docs/metadata-fields.toml`, `tools/inbox-v2-cost/guest/Cargo.toml`, `tools/inbox-v2-cost/guest/src/lib.rs`, `tools/inbox-v2-cost/host/Cargo.toml`, `tools/inbox-v2-cost/host/src/main.rs`, `published-contract/candidate/README.md`, `published-contract/candidate/inbox_v2_code_hash`, `scripts/check-inbox-vectors.sh (outside owned paths)`, `scripts/check-inbox-candidate.sh (outside owned paths)`, `scripts/check-inbox-cost.sh (outside owned paths)`, `scripts/check-fdev-verify-merge.sh (outside owned paths)`, `scripts/check-branch-coverage.sh (outside owned paths)`, `scripts/check-mutants-triage.sh (outside owned paths)`, `scripts/build-wasm.sh (outside owned paths)`, `scripts/check-excluded-crates.sh (outside owned paths)`, `scripts/check-guards-fail-inbox.sh (outside owned paths)`, `.github/workflows/inbox-v2-gates.yml (outside owned paths)`, `.github/workflows/inbox-v2-laws.yml (outside owned paths)`, `.release-freeze (outside owned paths)`

Skimmed: `contracts/inbox-v2/tests/fixtures/vectors.txt (vector names and byte lengths only)`, `contracts/inbox-v2/tests/fixtures/ghost-keys.txt (line shapes and tiers only)`, `contracts/inbox-v2/Cargo.lock (grepped for selected package versions)`, `published-contract/candidate/inbox_v2 (first 48 bytes only, to confirm the 40-byte header)`, `protocol/src/inbox_v2.rs (lines 1-80 and 400-470)`, `delegates/mail-delegate/src/inbox_v2.rs (excerpts around decisions 6, 18, 23)`, `delegates/mail-delegate/src/inbox_jobs.rs (lines 66-95)`, `delegates/mail-delegate/src/tests/inbox_jobs.rs (lines 1595-1625)`, `delegates/mail-delegate/Cargo.toml (inbox-v2 dependency lines)`, `ui/src/v2/mod.rs (lines 15-40, 120-150)`, `ui/Cargo.toml (test-ghost-master feature lines)`, `.github/workflows/check-contract-wasm.yml (inbox v2 steps, lines 255-310)`, `scripts/check-guards-fail.sh (inbox guard names and runner functions)`, `scripts/select-guards.sh (inbox area lines)`, `scripts/check-branch-coverage.py (docstring)`, `scripts/check-fdev-verify-merge_test.sh (lines 1-40)`, `scripts/check-wasm-builds.sh (crate list)`, `scripts/check-release-freeze.sh (header)`, `scripts/check-release-delegate.sh (candidate lines)`, `published-contract/README.md (lines 70-110)`, `README.md (headings and keyword search)`, `Makefile.toml (test tasks, inbox-v2 e2e task names)`, `Cargo.toml (workspace exclude lines)`, `rust-toolchain.toml`, `contracts/.cargo/config.toml`, `~/.cargo/registry ghostkey-verify-0.1.0/src/lib.rs and ghostkey-common-0.4.0 (API and verify path)`

Skipped: `tools/inbox-v2-cost/guest/Cargo.lock (lockfile)`, `tools/inbox-v2-cost/host/Cargo.lock (lockfile)`, `published-contract/candidate/mail_delegate and mail_delegate_code_hash (binary; belongs to the mail-delegate subsystem)`, `published-contract/candidate/inbox_v2 body (WASM binary)`
