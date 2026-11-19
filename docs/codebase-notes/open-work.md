> Notes v2, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# open-work: cross-cutting sweep (unfinished work, decisions, road to v0.3.0, stale docs)

Snapshot: upstream `2358b59` plus two docs-only commits of this fork (`83579e0`, `8b887ee`, adding `docs/codebase-notes/`). Paths are relative to the repo root. Nothing was built or run. Abbreviations: `C/` = `contracts/inbox-v2/`, `D/` = `delegates/mail-delegate/src/`, `P/` = `protocol/src/`.

## 1. Purpose and where this sits

A sweep, not a subsystem. It answers: what stands between this tree and a v0.3.0 release, what the numbered "decisions" in comments mean, which issues the tree cites, what is unfinished or dead, and where README, RELEASING, AGENTS and comments disagree with the code. Subsystem detail is in the sibling maps.

## 2. Walkthrough: the files that define the state of the project

- `.release-freeze:1-11`: plain-text reason for the freeze. Last edited in #368 (2026-11-02), before scrub, jobs and inbox v2 landed.
- `scripts/check-release-freeze.sh:27-31`: exit 3 while the file exists; `RELEASE_FREEZE_FILE` is honoured only with `RELEASE_FREEZE_TEST=1` (`:18-25`).
- `scripts/check-release-delegate.sh`: passes any version below 0.3.0 (`:66-69`). From 0.3.0 it fails if `ui/` contains the string `published-contract/candidate` (`:72-77`), if `published-contract/wasm/v2/mail_delegate` is missing (`:78-81`), if that file holds the marker `freenet-mail-delegate:v1-transition-enabled` (`:82-86`), or if it differs from a fresh `build-wasm.sh ... freenet-main-delegate` build (`:87-96`).
- `scripts/check-release-scrub.sh:32,39,45`: three WASM marker strings. `freenet-mail:scrub-built-in` must be present in a production build; `freenet-mail:e2e-probe-built-in` and `freenet-mail:test-ghost-master-built-in` must be absent, except for harness publishes run with `--absent` and `FREENET_E2E_PROBE_ISO=1` / `FREENET_TEST_GHOST_ISO=1`.
- `scripts/release.sh`: freeze gate first (`:37`), version match (`:140-146`), delegate check (`:152`), frozen pins (`:209`), `cargo make test`, `test-inbox`, `clippy` (`:216-218`), `test-e2e-real-node` unless `FREENET_RELEASE_E2E=0` (`:226-229`), `publish-production` (`:285`), pointer flip (`:321-322`).
- `published-contract/README.md:78-99`: the only written procedure for adding a generation. `published-contract/candidate/README.md:8-9`: what the release does with the candidates.
- `legacy_inbox.toml` (2 rows: pre-#253 and v0.2.5), `legacy_identity_delegate.toml` and `legacy_local_state_delegate.toml` (2 rows each: generation 205 = v0.2.5, 206 = v0.2.6/v0.2.7).
- `docs/qa/manual-test-inventory.md`: by status about 66 auto, 38 manual, 6 blocked, 1 fixme rows.
- `docs/codebase-notes/`: this fork's own reading notes (this directory); not upstream documentation.

## 3. Interfaces the release path depends on

- Cargo features: delegate `v1-transition` (`delegates/mail-delegate/Cargo.toml:62-64`), enabled by the candidate build (`scripts/check-delegate-candidate.sh:24`) and by the UI's in-page and test copies of the delegate (`ui/Cargo.toml:41,127`). UI features `use-node`, `example-data`, `no-sync`, `local-delegate`, `e2e-probe`, `test-ghost-master` (`ui/Cargo.toml:98-122`).
- Build-time environment: `FREENET_MAIL_SCRUB=1` (set only by `publish-production`, `Makefile.toml:861-870`) and `FREENET_UI_FEATURES` (`Makefile.toml:515-527`).
- Embedded artifacts today: frozen v1 (`ui/src/inbox.rs:57-58`, `ui/src/aft.rs:26-34`); candidates at `ui/src/mail_delegate/node.rs:20,22`, `ui/src/v2/mod.rs:27,29` and `D/inbox_jobs.rs:84`.
- Protocol surface tied to the transition: `Op::V1AftParams` (`P/lib.rs:319-324`), `Status.v1_transition` (`P/lib.rs:561`).

## 4. Rules, and the numbered design decisions

Release rules and where each is enforced:
- No publish while frozen: `check-release-freeze.sh`, tested by `scripts/release-freeze_test.sh` (`.github/workflows/build.yml:161`).
- A production build has scrub and no test instrumentation: `check-release-scrub.sh`, tested by `check-release-scrub_test.sh` (`build.yml:165`).
- 0.3.0 and later ship a frozen delegate without `v1-transition`: `check-release-delegate.sh`; its test runs only inside `scripts/check-guards-fail.sh:1754`.
- Nothing writes `published-contract/`: `check-frozen-artifacts.sh` (`release.sh:209`).

**Decisions.** Nothing in the tree defines "decision N", "the plan" (`plan 5.3/5.4/5.5`, `§5.5`), the reviews (`r1`-`r4`; findings `B1`, `M1`, `N1`, `N4`, `S2`), the packages `C1`/`C2`/`C3`, or the UI review tags `X6`, `Y6` ("round 9") and `R12`. Reconstructed from every citation:

| # | Meaning | Key citations |
|---|---|---|
| 1 | A Ghost Key proof signs exactly the `fmail/ghost-admit/v1` layout over (recipient, sent_time, ct_hash), for a pinned requestor | `P/inbox_v2.rs:287`, `C/tests/plausible_mutants.rs:301`, `ui/src/ghostkeys.rs:3`, `ui/src/v2/net.rs:1046` |
| 2 | Quarantine admission and caps, as amended; a proof is attached only for a RequireGhost recipient | `ui/src/v2/send.rs:184`, `ui/src/app/v2_view.rs:608`, `C/tests/oracle/mod.rs:27-29` |
| 3 | One message per id; the smallest encoding wins | `C/tests/oracle/mod.rs:28` |
| 4 | Owner mail is signed by the issuer key (`fmail/owner-msg/v1`) | `P/inbox_v2.rs:282` |
| 5 | Time is the highest `(owner_time, sig)` verifying under the joined issuer; caps keep the smallest ids | `C/tests/oracle/mod.rs:13,24` |
| 6 | The issuer key never changes; a record that changes it makes later merges refuse, pinned on purpose | `D/inbox_v2.rs:446`, `C/tests/tamper.rs:517` |
| 8 | A worst-case contract call stays under 2.5 s (core's limit is 5 s), gated as 37e9 wasmtime fuel | `scripts/check-inbox-cost.sh:2,6,29-30`, `.github/workflows/inbox-v2-gates.yml:164` |
| 9 | Drain before signing: nothing is signed that drops a message not yet stored. A filtered update is no error, so senders read back and reseal at `reseal_at` | `D/inbox_v2.rs:1671,2083`, `D/inbox_jobs.rs:7`, `P/inbox_v2.rs:331`, `P/lib.rs:384,534`, `ui/src/v2/send.rs:311` |
| 10 | Only revocations left after pruning filter mail; a set at its cap refuses every pass | `C/tests/rules.rs:407`, `C/tests/oracle/mod.rs:18-21`, `D/inbox_v2.rs:1539` |
| 11 | A message id binds the pass id (pass mail) or the proof's ghost key (quarantine), so a re-proved copy is a different message | `C/src/signed.rs:214`, `P/inbox_v2.rs:312`, `C/tests/ghost.rs:400-404` |
| 12 | Per-part bounds (88 messages, 4096 revocations, 1024 usage entries, 4096 tombstones, 4 MiB + 256 KiB) refused at decode | `C/src/types.rs:25-34`, `C/tests/bounds.rs:1` |
| 13 | Summary entries are 16 id bytes + 16 digest bytes | `C/src/sync.rs:18` |
| 14 | Quarantine mail may lead `owner_time` by 2 days, not 400; quarantine and per-ghost caps | `P/inbox_v2.rs:37`, `C/tests/rules.rs:66`, `C/tests/oracle/mod.rs:26,64` |
| 16 | The decision-12 bounds apply to the sum over all parts of one call | `C/src/types.rs:39`, `C/src/lib.rs:91`, `C/tests/bounds.rs:204` |
| 17 | Expiries are whole days; a pass lifetime is drawn from 335-395 days, so an expiry does not give the grant time | `C/src/merge.rs:67`, `D/inbox_v2.rs:23-27,52`, `P/inbox_v2.rs:40` |
| 18 | Previous-epoch grace: passes and revocations of epoch - 1 live while `owner_time < prev_epoch_until`; a routine bump sets 14 days, an emergency bump 0 | `C/src/types.rs:164`, `C/src/merge.rs:16,238`, `D/inbox_v2.rs:59,1526` |
| 19 | An Open inbox refuses any message carrying a proof | `C/src/merge.rs:353`, `C/tests/ghost.rs:300`, `D/inbox_v2.rs:2007`, `C/mutants-triage.md:50-58` |
| 20 | Map keys are public: each is a disclosure path `<map>.[key]`. Cited with 17 for linkability | `C/tests/metadata_disclosure.rs:16`, `D/inbox_v2.rs:23` |
| 21 | The delegate keeps the newest 1024 drained stranger messages, stored copies and tombstones | `D/inbox_v2.rs:67,1643`, `P/lib.rs:479`, `ui/src/v2/queue.rs:22` |
| 22 | Revocations of one pass id rank by `(epoch, expiry, sig)`; ranking expiry first was a real bug | `C/src/merge.rs:219`, `C/tests/rules.rs:1079`, `C/tests/merge_laws.rs:886` |
| 23 | The client bumps the epoch at 2048 live revocations; a full set needs an emergency bump; a revocation signs the largest expiry; `used` only grows | `C/src/merge.rs:247`, `D/inbox_v2.rs:61,309,314`, `C/tests/rules.rs:727` |
| 24 | Every v2 ML-DSA signature has a FIPS 204 context (`fmail/v2/owner`, `fmail/v2/sealed`); v1 signs with none | `C/src/signed.rs:54`, `P/inbox_v2.rs:27-31`, `D/crypto.rs:228`, `D/handlers.rs:1572` |
| 25 | The delegate notes (`foreign_seen`) an owner record it did not sign; the page warns "active on another node"; nothing clears the note | `D/inbox_v2.rs:228,267,1615`, `ui/src/v2/owner.rs:45`, `ui/src/app/v2_view.rs:24` |

Decisions 7 and 15 are never cited. `ui/src/app/v2_view.rs:99` says "decision D8" for the Open/RequireGhost choice, which is not decision 8 above: a second numbering or a mislabel. `C1` is the contract package (`D/inbox_v2.rs:1806`). `C2` is where the envelope is built, the delegate (`C/src/signed.rs:168`). `C2c` resolves a legacy 44-character address to a v2 address (`ui/src/address.rs:76-77`). `C3` is a later package in which `release.sh` gains the metadata-disclosure gate (AGENTS.md: "comes with C3").

## 5. State: the v0.3.0 chain

Landed since the v0.2.7 release commit `3455182` (git log): #353 frozen artifacts and registries, #359 stdlib 0.12.1, #364 mail delegate and protocol, #366 UI on the mail delegate, #368 migration walk, #369 scrub, #370 background jobs, #372/#374/#378 inbox v2 contract, delegate operations and UI, and #371/#375/#380 CI guards. The workspace version is still `0.2.7` (`Cargo.toml:56`).

The freeze file's four "still to land" items against the tree:

| Item (`.release-freeze:5-7`) | In tree | Evidence |
|---|---|---|
| scrub | yes (#369) | `ui/src/migration/scrub.rs`; marker gate `check-release-scrub.sh:32` |
| background jobs | yes (#370) | `D/jobs.rs`, `D/watch.rs`, `ui/src/jobs.rs` |
| inbox v2 | yes | `contracts/inbox-v2/`, `D/inbox_v2.rs`, `ui/src/v2/` |
| rehearsal against real v0.2.x data | no | "rehears" occurs only in `.release-freeze:6,8`. Nearest: `cargo make test-e2e-legacy-migration` (data seeded by `tools/v1-node-seed`) and `ui/src/migration/captured_v0_2_7.txt` |

What blocks `scripts/release.sh 0.3.0`, in the order it would fail:

1. `.release-freeze` exists. The gate is at `release.sh:37` and is the first dependency of `publish-container`, `publish-email` and `publish-production` (`Makefile.toml:436,837,861`).
2. The version is 0.2.7 (`release.sh:140-146`).
3. `published-contract/wasm/` holds only `v1`: there is no `wasm/v2/mail_delegate`.
4. `ui/` references candidates in three places: `ui/src/mail_delegate/node.rs:20,22`, `ui/src/v2/mod.rs:27,29` and `ui/src/legacy_registry.rs:143` (a test). The grep covers the whole `ui/` directory, so the inbox v2 includes count too.
5. The delegate itself includes `candidate/inbox_v2_code_hash` (`D/inbox_jobs.rs:84`; also `D/tests/jobs.rs:176`). No script checks this. Order therefore matters: freeze inbox v2, repoint the delegate, then build and freeze the delegate.
6. The frozen delegate must be built without `v1-transition`; the committed candidate is built with it (`published-contract/candidate/README.md:13,24`). The v0.3.0 delegate therefore has a different code hash, and so a different delegate key, from every candidate so far. Data stored under a candidate key by anyone running main does not carry over, and no importer from one mail-delegate generation to another exists (delegate-core map, section 10).
7. The v1 send path needs `V1AftParams`. `ui/src/aft.rs:474-484` and `ui/src/api.rs:1268-1280` get the token-generator parameters (the identity's ML-DSA seed) from the delegate, and without the feature the handler answers `Refused "this build has no v1 send path"` (`D/handlers.rs:1597-1600`). Comments tie removal to "when the v2 inbox lands" and "with the anti-flood tokens" (`delegates/mail-delegate/Cargo.toml:62-63`, `ui/src/mail_delegate/ops.rs:529,537`, `P/lib.rs:319-321`). Inbox v2 has landed, yet v1 send, AFT and the v1 inbox client are all still live, compose still routes every non-v2 recipient to v1 (ui-v2 map), and a v1 contact cannot become a v2 contact (`ui/src/v2/contacts.rs:181-187`). This is the largest piece of real work left, and the tree holds no plan for it.
8. Pins and rows: move the candidates to `wasm/v2/`, add them to `frozen.sha256` (`candidate/README.md:8-9`), repoint `ui/src/legacy_registry.rs:143`.
9. `release.sh` then runs the full test gate and, by default, `test-e2e-real-node`.

Named by the freeze file but required by no script: the rehearsal. Gated by nothing: `cargo make test-e2e-inbox-v2` runs in no workflow (the workflows run only `test-e2e-background`, `test-e2e-real-node` and `test-e2e-legacy-migration`), so inbox v2 on real nodes is proven only by hand.

## 6. Tests of the release gates

| Command | Proves | Needs |
|---|---|---|
| `scripts/release-freeze_test.sh` | the gate exits 3; the override needs the test flag; `release.sh` runs the gate before parsing arguments; three publish tasks list it first | bash |
| `scripts/check-release-scrub_test.sh` | marker logic for scrub, probe and ghost master, and that the publish tasks call the check | bash |
| `scripts/check-release-delegate_test.sh` | version cut-off, candidate grep, marker, byte comparison | `fdev` and the wasm target (it builds the delegate) |
| `scripts/check-guards-fail.sh --list` / `--anchors` | prints the guards; checks that every mutation still applies, without building | bash |

## 7. Changing it safely: traps on the way to v0.3.0

- Removing `.release-freeze` probably breaks a guard. The guard "the freeze override needs the test flag" (`scripts/check-guards-fail.sh:1726-1728`) expects the line "RELEASE_FREEZE_FILE without the test flag does not lift the freeze", which `scripts/release-freeze_test.sh:40-46` emits only `if [ -e "$ROOT/.release-freeze" ]`; the guard's staging copy is built from `git ls-files` (`check-guards-fail.sh:76`). Without the file, the mutation cannot fail a test, and a guard that does not fail turns `check-drift` red. Rewrite or retire the guard in the same change.
- An inbox v2 source change forces both candidates to be rebuilt (the delegate embeds the contract's code hash); the candidate README does not say so.
- Removing `V1AftParams` touches `protocol/`: the golden file is append-only and `P/golden.rs:36,239` pins the op, so the variant stays on the wire even when no build serves it.
- `RELEASING.md` needs a "freeze the candidates" section; that step exists only in `published-contract/README.md:78-99`.
- A release publishes to the fixed production contract id. The first 0.3.0 page load runs the migration walk and, with `FREENET_MAIL_SCRUB=1`, scrub, against real users' v0.2.x delegates. That is what the rehearsal was meant to cover.

## 8. Disagreements between docs/comments and code

README.md (last edited in #353, before the mail delegate existed):
- `:31-33,70-71` link issues at `iduartgomez/freenet-email`; comments elsewhere say `freenet/mail` (`scripts/check-frozen-artifacts.sh:6`, `scripts/check-wasm-paths.sh:7`).
- `:40-42,77-80`: keys "live inside the identity delegate (`modules/identity-management/`)". That delegate is retired; keys are in `delegates/mail-delegate`.
- `:49`: "Three on-chain contracts and one delegate", then five bullets; no mail delegate, inbox v2, Ghost Keys, migration or background jobs.
- `:44-45`: a send burns an Anti-Flood Token only on the v1 path.
- `:94-96`: AGENTS.md has no "feature-flag matrix".

RELEASING.md:
- `:224`: `cargo make test-ui-live` does not exist (the task is `test-e2e-real-node`, `Makefile.toml:1068`); the same name is in `scripts/smoke-test-production.sh:33-34,92-93`.
- `:226` "real RSA keygen"; `:218` "identity-management contracts".
- `:206`: the header is "Freenet Email"; the brand is "Mail" (`ui/branding/app.json:2`).
- `:202` "all five browser profiles" (and AGENTS.md's "5 browser/viewport profiles"): `ui/tests/playwright.config.ts:31-43` defines chromium and mobile-chrome only.
- `:231` "7-step checklist": AGENTS.md's has nine steps.
- `:149-153` omits `check-release-delegate.sh` and the default-on real-node run; the freenet-mail-release skill does too.

AGENTS.md:
- "(gated on `FREENET_LIVE_E2E_SEND=1`)": the gate was removed (`ui/tests/live-node.spec.ts:445-448,740`).
- "grep for `UPDATE_PROPAGATION`": current core never emits it; the spec polls `phase="update_complete"`, non-fatally (`live-node.spec.ts:547-556`).
- "intended for tags / nightly, not every PR": `.github/workflows/e2e-real-node.yml:12-18` runs on every pull request; that file's own header (`:3-6`) repeats the stale claim.
- "`ui/src/api.rs` (~1442 and ~2540)": the prime GETs are at `ui/src/api.rs:430` and `:2277`.
- "`prev_app_ids` keeps a small ring ... for client-side rollback": `sign-container-state` passes no previous ids (`Makefile.toml:358-386`) and the loader never reads them.
- "Append the previous hash to the slice (and to `legacy_inbox.toml`)": the two lists already differ (section 9, item 2).

Others:
- `.release-freeze:5-7`: three of four items have landed.
- `published-contract/candidate/README.md:15`: "`inbox_v2` ... Nothing embeds it yet" (last edited in #372; #374 and #378 embedded it).
- `docs/qa/manual-test-inventory.md:232-236,263-265` and `.claude/skills/freenet-mail-qa/SKILL.md:58-83` describe a `wip-settings` cargo feature. None exists (`ui/Cargo.toml:98-122`) and no source mentions it.
- `ui/src/legacy_registry.rs:8-9`: "The inbox list is not read at runtime yet"; `ui/src/migration/mod.rs:413-421` reads it.
- `ui/src/aft.rs:23-25` and `ui/src/inbox.rs:50-55`: hashes "produced by `cargo make build-token-*` / `build-inbox-contract`"; they are frozen files. `contracts/inbox/src/lib.rs:2-6` names a nonexistent `build-inbox` task and a workspace build of a crate the workspace excludes.
- `ui/src/lib.rs:109-113,153`, `release.sh:347` and `tools/web-container-sign/tests/facade_integration.rs:71-88`: "Phase 1 prereq / no committed snapshot" wording, though the container snapshot is committed.
- `ui/src/api.rs:3447-3449,3582`: "dead until the Phase 1 restore lands", about the retired identity delegate.
- `ui/src/app.rs:1976`: "production does not yet fold Sent into the inbox thread", against commit `2f3ba4c` (#310/#311). Not verified in code.
- `ui/src/app/login.rs:249`: `#[allow(dead_code)]` on `ml_dsa_vk`, which `ui/src/aft.rs:482` calls.

## 9. Suspected bugs and weak spots

1. **The freeze guard depends on the freeze file** (section 7). Medium-high confidence; not run.
2. **Two legacy inbox lists disagree.** `ui/src/inbox.rs:71-76` has one hash; `legacy_inbox.toml:17-25` has two (it adds v0.2.5's `9Kuhod...`). The v1 inbox migration walks the one-entry slice (`ui/src/api.rs:946`). The recorded hash is always tried first (`ui/src/inbox.rs:90-101`), so a user stamped with `9Kuhod...` still migrates; one stamped with an older hash whose mail sits under `9Kuhod...` would not find it. Certain as an inconsistency, low as a live bug.
3. **`todo!()` on node disconnect** (`ui/src/api.rs:2697-2699`) panics the node loop in every `use-node` build. High, if core sends `RequestError::Disconnect`.
4. **Other `ContractError`s are logged as `"FIXME: {err}"` and dropped** (`ui/src/api.rs:2669-2671`), and a failed UPDATE on an own token record does not reject the pending assignment (`:2597-2598`). High.
5. **`check-release-delegate.sh` never looks for a frozen inbox v2**, and nothing checks that the delegate's embedded inbox hash equals the UI's. Both read one candidate file today; after the freeze they are two includes that can drift. Medium.
6. **No importer between mail-delegate generations**, while section 5 item 6 guarantees a key change at 0.3.0. Harmless for production users, who arrive from v0.2.x; it loses data for anyone testing main. High.
7. `.gitignore:5` ignores `*.rs.bk`, not `.bkp`, which is how the stray file in section 10 stays tracked. Trivial.

## 10. Open work

**Every TODO-class marker** (15 hits; none in the mail delegate, `protocol/`, inbox v2 or `ui/src/v2/`):

| Where | Meaning |
|---|---|
| `ui/src/api.rs:2698` | `todo!("lost connection to node, should retry connecting")` |
| `ui/src/api.rs:2590,2597,2670` | three FIXMEs in node error handling (section 9, item 4) |
| `ui/src/api.rs:42,186` | `unimplemented!()` for cfg combinations that never run |
| `ui/src/inbox.rs:545,549` | dead placeholders `StoredDecryptedSettings` and `InboxSummary` |
| `ui/src/inbox.rs:689` | `todo:` no confirmation tracking after a v1 UPDATE send |
| `ui/src/lib.rs:46` | `MAIN_ELEMENT_ID` unused |
| `contracts/inbox/src/lib.rs:1002` | the v1 inbox never sets `last_update`; frozen, so unfixable without a new generation |
| `modules/antiflood-tokens/contracts/token-allocation-record/tests/test.rs:6-10` | `TODO(phase-0)`: the only integration test is `#[ignore]`d for a fixture nothing builds |
| `ui/tests/live-node.spec.ts:2258,2281` | `test.fixme` and `TODO(#291)`: delete and regenerate an identity (#5) |
| `.../token-allocation-record/src/lib.rs:167,384` | historical mentions of a FIXME that #71 fixed |

**"Not yet" without a marker:**
- v1 contacts "move over in a later update" (`ui/src/v2/contacts.rs:183-185`).
- `export_profile` has no UI (`ui/src/mail_delegate/ops.rs:554-555`).
- `Job::SendQueued` is listed and unimplemented (`D/jobs.rs:132,153`).
- `SealedBody.pass_grant` is read (`D/inbox_v2.rs:1480`) and always sealed as `None` (`:2017`).
- True unarchive (#60, `ui/src/local_state/items.rs:563`); read receipts (#69), `pad_length`, `display_name` (inventory `:263-265`); AFT tier configurability (#85, inventory `:283`); contact key auto-discovery (#42); an in-protocol inbox upgrade pointer (#251 improvement 1, AGENTS.md).
- Loader Phase 2 with rollback (`contracts/facade-loader/README.md:32-36`); the metadata gate in `release.sh` (C3).

**Documented limits** beyond AGENTS.md's own "Documented limits" paragraph:
- Scrub leaves settings and most read marks in the old delegates and never touches the token generators (`ui/src/migration/scrub.rs:51-54`); a check that can never pass leaves the old copy for good (`scrub.rs:7-8`).
- Items under an unreadable list survive identity deletion as unreachable bytes (`D/tests/faults.rs:807-811`).
- The offline v2 model checks no signatures, expiries, epochs or caps (`ui/src/v2/net.rs:820`).
- Cross-node delivery tests skip themselves when core's relay bug signature appears (core #3279/#3465, `ui/tests/live-node.spec.ts:583-604`); cross-node threading is asserted as a known limitation (#270, `live-node.spec.ts:955-993`).
- Own-inbox self-heal cannot be triggered on the two-node net (inventory `:282`).
- Inbox v2 is untested anywhere for pass renewal, the real ghostkeys delegate, the vault's return link and the strangers-dropped count (inventory `:56-63`).

**Issue numbers cited (about 130 distinct).** A bare `#NNN` means the project's own tracker; only README gives URLs. By theme, with what the tree says:
- Build and release: #198 and #199 (lockfile isolation; "Phase B" is the migration story), #200/#205/#206 (container contract; snapshot canonical and gated), #214 and #241 (parameter tripwires), #339 (`check-drift` on all but docs-only PRs), #348 (builds embed machine paths, hence frozen bytes), #9 ("Phase 5" release workflow; `release.sh:372` still says to comment on it), #6, #41.
- v1 inbox and migration: #213, #223, #251 (improvements 2-5 shipped, 1 open), #253 (signed payload widened; row 0 of `legacy_inbox.toml`), #254/#255/#256/#260, #288 (own inbox loaded by GET with subscribe; self-heal), #286 (deletes persist).
- AFT: #71/#72, #85 (tiers; the day-1 cap is still blocked), #150/#157 (verified-sender bypass), #152, #160, #174, #179, #180, #221/#273, #266/#267/#271.
- UI: #249 (share card and fingerprint), #270/#283/#287/#310/#317 (threads), #289 (reply by key), #290/#291 (send-routing guard; deferred real-node tests), #295/#302, #349 (recovery key), #106/#107/#113 (optimistic read and archive state).
- Open or blocked, per the tree: #5, #42, #60, #69, #85, #291.
- freenet-core: #3279/#3465, #3980, #4003, #4007, #4008, #4071, #4077 (AFT `fetch_related` timeout), #4102 (fdev 300 s timeout), #4146, #4345 (silent UPDATE drop), #6244 (CI concurrency).
- "#1" and "#3 in the AFT audit" (`ui/src/aft.rs:279`, `wire-v1/src/aft.rs:682`) are audit item numbers, not issues. No issue is cited for v0.3.0 itself or for any step of section 5.

**Leftover files and tasks:**
- `modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp`: 8 KB, RSA-era, never compiled, the only file in that `tests/` directory.
- Makefile tasks that build nothing shipped: `build-inbox-contract`, `build-token-*`, `build-*-delegate`, `generate-*-params`, `build-contracts`, `build-web-container` (`Makefile.toml:19-226`; the section comment says so).
- Twelve alias tasks for old names (`Makefile.toml:494-513,727,770`).
- Committed PEM keys for the retired delegates (`test-contract/identity/`, `test-contract/mail-local-state/`); only the frozen parameters derived from them still matter.

## 11. Questions the tree does not answer

- Where are the decisions, "the plan" and reviews r1-r4 written down? What were decisions 7 and 15?
- What is the "rehearsal against real v0.2.x data": whose data, on which node, with what pass criteria?
- Is v1 sending removed in 0.3.0, kept read-only, or is the `v1-transition` guard to be relaxed? How do v1 contacts become v2 contacts (C2c)?
- Users on v0.2.0-v0.2.4: the delegate registries start at generation 205 (v0.2.5). Did earlier releases ship other delegate bytes, and are those users stranded?
- Does anyone outside the developers run main builds against the real network (candidate inbox v2 contracts under the production Ghost Key master)?
- Which tracker do bare `#NNN` references resolve in today, given README's links and the move to `freenet/mail`?

---

## Appendix: the reader's structured summary

Cross-cutting sweep of the whole repo at upstream 2358b59 (plus two docs-only fork commits). Nothing was built or run. The map is complete on disk (3,355 words, all 11 sections), so a restart does not need to redo this unit.

State: the tree is between releases. Workspace version is 0.2.7; ten feature/CI PRs for v0.3.0 sit on top (#353 to #380). Releases are blocked by .release-freeze, whose text is stale: of its four "still to land" items, scrub, background jobs and inbox v2 are in the tree; only the "rehearsal against real v0.2.x data" is not, and nothing in the tree describes one.

What really blocks 0.3.0: no published-contract/wasm/v2/; the UI and the delegate still include candidate artifacts; the frozen delegate must be built without v1-transition, while the v1 send path (still live, still the route for every non-v2 recipient) needs the V1AftParams op that feature provides. Removing or reworking v1 sending is the largest remaining job and has no plan in the tree.

The numbered design decisions (1-25, 7 and 15 never cited) are defined nowhere; the map reconstructs each from its citations. TODO-class markers are few (15) and all in v1-era code; new code states open work as documented limits. README.md predates the mail delegate and is wrong about where keys live and what the system consists of; RELEASING.md and AGENTS.md carry several stale commands and claims. One likely trap: deleting .release-freeze appears to break a required guard.

### Key facts

- Workspace version is 0.2.7 (Cargo.toml:56); scripts/release.sh:140-146 refuses a version mismatch, so 0.3.0 needs a bump commit first.
- .release-freeze:5-7 lists scrub, background jobs, inbox v2 and a rehearsal as still to land; the file was last edited in #368 and the first three have since landed (#369, #370, #372/#374/#378). 'rehears' occurs nowhere else in the tree.
- scripts/check-release-freeze.sh:27-31 exits 3 while the file exists; it is run first by scripts/release.sh:37 and is the first dependency of publish-container, publish-email, publish-production (Makefile.toml:436,837,861).
- scripts/check-release-delegate.sh:66-96: from 0.3.0 it fails if ui/ contains 'published-contract/candidate', if published-contract/wasm/v2/mail_delegate is missing, if it carries the v1-transition marker, or if it differs from a fresh build without that feature.
- published-contract/wasm/ holds only v1. Candidates are embedded at ui/src/mail_delegate/node.rs:20,22, ui/src/v2/mod.rs:27,29 and referenced by a test at ui/src/legacy_registry.rs:143.
- The delegate itself includes the candidate inbox v2 code hash (delegates/mail-delegate/src/inbox_jobs.rs:84); no release script checks that path. Freeze order must be inbox v2 first, then the delegate.
- The committed delegate candidate is built WITH v1-transition (scripts/check-delegate-candidate.sh:24); the release delegate must be built without it, so its code hash and delegate key will differ from every candidate so far.
- The v1 send path depends on Op::V1AftParams (ui/src/aft.rs:474-484, ui/src/api.rs:1268-1280); without the feature the handler answers Refused (delegates/mail-delegate/src/handlers.rs:1597-1600). v1 send, AFT and the v1 inbox client are all still live.
- v1 contacts cannot be added as v2 contacts: ui/src/v2/contacts.rs:181-187 tells the user 'Old contacts move over in a later update'; ui/src/address.rs:76-77 calls the missing piece 'C2c'.
- scripts/release.sh runs, beyond what RELEASING.md lists, check-release-delegate.sh (:152) and by default cargo make test-e2e-real-node (:226-229, opt out with FREENET_RELEASE_E2E=0).
- Production builds must carry the marker 'freenet-mail:scrub-built-in' and must not carry the e2e-probe or test-ghost-master markers (scripts/check-release-scrub.sh:32,39,45); FREENET_MAIL_SCRUB=1 is set only by publish-production (Makefile.toml:861-870).
- cargo make test-e2e-inbox-v2 (Makefile.toml:1133) runs in no workflow; the workflows run only test-e2e-background, test-e2e-real-node and test-e2e-legacy-migration (.github/workflows/e2e-background.yml:167, e2e-real-node.yml:165,181).
- The numbered design decisions, 'the plan', reviews r1-r4 and packages C1/C2/C3 are defined nowhere in the tree; the map's section 4 gives a table of decisions 1-25 with citations. Decisions 7 and 15 are never cited; ui/src/app/v2_view.rs:99 uses a different label 'D8'.
- Only 15 TODO/FIXME/todo!/unimplemented! hits exist in tracked source, none in the mail delegate, protocol/, inbox v2 or ui/src/v2/. The live one that matters: ui/src/api.rs:2698 todo!() on RequestError::Disconnect.
- The legacy registries start at v0.2.5: legacy_identity_delegate.toml and legacy_local_state_delegate.toml each have rows for generation 205 and 206 only; legacy_inbox.toml has 2 rows while the in-source slice ui/src/inbox.rs:71-76 has 1.
- README.md was last edited in #353, before the mail delegate (#364); its issue links point at iduartgomez/freenet-email (README.md:31-33,70-71) while code comments name freenet/mail (scripts/check-frozen-artifacts.sh:6).
- The guard 'the freeze override needs the test flag' (scripts/check-guards-fail.sh:1726-1728) relies on a test branch that runs only while .release-freeze exists (scripts/release-freeze_test.sh:40-46).
- Dead surface in new code: Job::SendQueued has no implementation (delegates/mail-delegate/src/jobs.rs:132,153); SealedBody.pass_grant is read at delegates/mail-delegate/src/inbox_v2.rs:1480 and always sealed None (:2017); export_profile has no caller (ui/src/mail_delegate/ops.rs:554-561).
- A stray uncompiled file is tracked: modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp; .gitignore:5 ignores only *.rs.bk.
- The Playwright config defines two projects, chromium and mobile-chrome (ui/tests/playwright.config.ts:31-43), not the five that AGENTS.md and RELEASING.md describe.

### Docs or comments that disagree with the code

- **[high]** Claim (.release-freeze:5-7): Scrub, the background jobs, inbox v2 and the rehearsal are still to land **Reality:** Scrub (#369), background jobs (#370) and inbox v2 (#372, #374, #378) are in the tree. Only the rehearsal is missing, and nothing else in the tree mentions one. Evidence: ui/src/migration/scrub.rs:1; delegates/mail-delegate/src/jobs.rs:49; ui/src/v2/mod.rs:27
- **[medium]** Claim (published-contract/candidate/README.md:15): inbox_v2: 'Nothing embeds it yet' **Reality:** The UI embeds the candidate WASM and its hash, and the delegate embeds its code hash, so an inbox v2 change forces a delegate candidate rebuild too. Evidence: ui/src/v2/mod.rs:27-29; delegates/mail-delegate/src/inbox_jobs.rs:84
- **[high]** Claim (README.md:40-42,49,77-80): Private keys live inside the identity delegate (modules/identity-management/); the system is 'three on-chain contracts and one delegate' **Reality:** The UI no longer runs the identity delegate; keys live in delegates/mail-delegate. README does not mention the mail delegate, inbox v2, Ghost Keys, migration or background jobs. README was last edited in #353. Evidence: ui/src/mail_delegate/node.rs:20
- **[low]** Claim (README.md:31-33,70-71): Issue links (#198, #200, #206, #213, #223) resolve at github.com/iduartgomez/freenet-email **Reality:** Upstream is freenet/mail, and code comments already cite 'freenet/mail#348'. No other file links to iduartgomez. Evidence: scripts/check-frozen-artifacts.sh:6
- **[low]** Claim (README.md:94-96): AGENTS.md holds a feature-flag matrix **Reality:** AGENTS.md has no such matrix; it points at ui/Cargo.toml for the feature list. Evidence: ui/Cargo.toml:98-122
- **[medium]** Claim (RELEASING.md:224; scripts/smoke-test-production.sh:33-34,92-93): `cargo make test-ui-live` runs the live-node spec **Reality:** No such task exists. The task is test-e2e-real-node. Evidence: Makefile.toml:1068
- **[low]** Claim (RELEASING.md:218,226): The live test exercises 'real RSA keygen' and needs 'inbox, AFT, and identity-management contracts published' **Reality:** Identities are ML-DSA-65/ML-KEM-768 and identity-management is a retired delegate, not a contract. Evidence: Cargo.toml:79-84
- **[low]** Claim (RELEASING.md:206): The smoke test checks that Dioxus renders the 'Freenet Email' header **Reality:** The brand name is 'Mail', read from ui/branding/app.json by both the UI and the spec. Evidence: ui/branding/app.json:2
- **[medium]** Claim (AGENTS.md 'Automated (Playwright)'; RELEASING.md:202): The Playwright suite runs across five browser/viewport profiles **Reality:** The config defines two projects, chromium and mobile-chrome; Firefox and WebKit are commented as omitted. Evidence: ui/tests/playwright.config.ts:31-43
- **[high]** Claim (RELEASING.md:149-153): Preflight is branch, tree, tag, key, tools, node, frozen artifacts; test gate is test, test-inbox, clippy **Reality:** release.sh also runs check-release-delegate.sh and, by default, cargo make test-e2e-real-node. RELEASING.md never mentions the delegate check or freezing the candidates. Evidence: scripts/release.sh:152
- **[low]** Claim (RELEASING.md:231; scripts/smoke-test-production.sh:36): The manual acceptance gate is a '7-step checklist' in AGENTS.md **Reality:** AGENTS.md's manual E2E checklist has nine steps. Evidence: RELEASING.md:231
- **[low]** Claim (AGENTS.md 'Automated real-node harness'): The send round trip in the live spec is gated on FREENET_LIVE_E2E_SEND=1 **Reality:** The gate was removed; the spec's own comments say so. Evidence: ui/tests/live-node.spec.ts:445
- **[low]** Claim (AGENTS.md 'Automated real-node harness'): Log assertions grep the gateway log for UPDATE_PROPAGATION and allocate_token **Reality:** The spec says current core never emits UPDATE_PROPAGATION; it polls for phase="update_complete" and the check is non-fatal. Evidence: ui/tests/live-node.spec.ts:547
- **[medium]** Claim (AGENTS.md 'CI scope'; .github/workflows/e2e-real-node.yml:3-6): The real-node target is heavy and intended for tags / nightly, not every PR **Reality:** The workflow triggers on every pull_request (#204(D)), as its own later comment says. Evidence: .github/workflows/e2e-real-node.yml:12
- **[low]** Claim (AGENTS.md 'Sender-side recipient hash'): The rehydrate and CreateContact prime GETs are in ui/src/api.rs at about 1442 and 2540 **Reality:** inbox_key_for_with_hash is called at lines 430 and 2277. Evidence: ui/src/api.rs:430
- **[medium]** Claim (AGENTS.md 'Chained migration'): Append the previous hash to the LEGACY_INBOX_CODE_HASHES slice and to legacy_inbox.toml **Reality:** The two lists already differ: the slice has one hash, the registry two (it adds the v0.2.5 hash 9Kuhod...). Evidence: ui/src/inbox.rs:71
- **[low]** Claim (AGENTS.md 'Container contract'): prev_app_ids keeps a small ring of previous pointers for client-side rollback; each release signs FacadePointer with prev_app_ids=[...] **Reality:** sign-container-state passes no previous ids, and the loader template never reads them. Rollback is a deferred 'Phase 2'. Evidence: Makefile.toml:358
- **[medium]** Claim (docs/qa/manual-test-inventory.md:232-236; .claude/skills/freenet-mail-qa/SKILL.md:58-83): WIP settings are gated behind the `wip-settings` cargo feature **Reality:** No such feature exists in ui/Cargo.toml and no source file mentions it. Evidence: ui/Cargo.toml:98
- **[low]** Claim (ui/src/legacy_registry.rs:8-9): The inbox registry list 'is not read at runtime yet' **Reality:** migration::v1_inbox_hashes reads it at runtime. Evidence: ui/src/migration/mod.rs:414
- **[low]** Claim (ui/src/inbox.rs:50-55; ui/src/aft.rs:23-25): inbox_code_hash and the AFT code hashes are 'produced by cargo make build-inbox-contract / build-token-*' **Reality:** They are include_str! of the frozen files under published-contract/wasm/v1/. contracts/inbox/src/lib.rs:5 also names a nonexistent `cargo make build-inbox`. Evidence: ui/src/inbox.rs:57
- **[low]** Claim (ui/src/lib.rs:109-113,153; ui/src/api.rs:3447-3449,3582; scripts/release.sh:347): The container id is 'empty when the snapshot hasn't been committed yet (Phase 1 prereq)'; the identities reload path is 'dead until the Phase 1 restore lands' **Reality:** The container snapshot is committed, and the identity delegate path was replaced by the mail delegate, so neither 'Phase 1' item is pending. Evidence: ui/src/lib.rs:153
- **[low]** Claim (AGENTS.md 'Gates on the rules'): release.sh refusing to publish on the metadata-disclosure list 'comes with C3' **Reality:** Accurate as 'not yet': release.sh has no such gate. But C3 is defined nowhere in the tree. Evidence: scripts/release.sh:152

### Suspected issues (leads from reading, not confirmed)

- **Deleting .release-freeze probably breaks a required guard** (confidence: medium). The guard 'the freeze override needs the test flag' expects the test line 'RELEASE_FREEZE_FILE without the test flag does not lift the freeze'. release-freeze_test.sh emits that line only inside `if [ -e "$ROOT/.release-freeze" ]`, and the guard's staging copy is built from git ls-files. Once the file is removed for v0.3.0 the mutation cannot fail any test, and a guard that does not fail turns check-drift red. Not run. Evidence: scripts/check-guards-fail.sh:1726
- **The v0.3.0 delegate will have a different key from every candidate, with no importer** (confidence: high). check-release-delegate.sh requires the frozen delegate to equal a build without v1-transition; the candidate is built with it. The delegate key is blake3 of the code hash, so anything stored by people running main under a candidate key is orphaned at 0.3.0. Production users come from v0.2.x and are not affected. Evidence: scripts/check-release-delegate.sh:82
- **v1 send path cannot work with the release delegate** (confidence: high). The UI gets the token-generator parameters through Op::V1AftParams; a delegate built without v1-transition answers Refused. As the tree stands, a 0.3.0 that passed the release guard could not send to any v1 recipient, and v1 contacts cannot be converted to v2. Evidence: delegates/mail-delegate/src/handlers.rs:1597
- **Two legacy inbox hash lists disagree** (confidence: low). The in-source slice walked by the v1 inbox migration has one hash; legacy_inbox.toml has two (adds the v0.2.5 inbox). The recorded hash is always tried first, so most users are unaffected; a user stamped with an older hash whose mail sits under the v0.2.5 hash would not be found. Certain as an inconsistency, low as a live bug. Evidence: ui/src/inbox.rs:71
- **Node disconnect panics the node loop** (confidence: high). RequestError::Disconnect is handled with todo!("lost connection to node, should retry connecting"). There is no reconnect. Reachable in every use-node build if core sends that error. Evidence: ui/src/api.rs:2698
- **Contract errors are logged and dropped** (confidence: high). Any ContractError other than the Update/Get cases is logged as the literal 'FIXME: {err}'. A failed UPDATE on an own token record only logs and toasts; the pending assignment is not rejected, as the FIXME beside it says. Evidence: ui/src/api.rs:2669
- **The release check does not cover a frozen inbox v2 or hash agreement** (confidence: medium). check-release-delegate.sh looks only for wasm/v2/mail_delegate and greps only ui/. Nothing requires wasm/v2/inbox_v2 to exist, and nothing checks that the inbox hash the delegate embeds equals the one the UI embeds. Both read one candidate file today; after the freeze they are separate includes that can drift. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:84
- **Inbox v2 on real nodes is not in CI** (confidence: high). cargo make test-e2e-inbox-v2 is defined but no workflow runs it, so delivery by pass, invite, queue, revocation and the closed-tab drain on real nodes are proven only when someone runs it by hand. Evidence: Makefile.toml:1133

### Open work

- Lift the freeze: .release-freeze:1-11 (and fix the guard at scripts/check-guards-fail.sh:1726-1728 in the same change).
- Freeze the candidates into published-contract/wasm/v2/, pin them in frozen.sha256, repoint ui/src/mail_delegate/node.rs:20,22, ui/src/v2/mod.rs:27,29, ui/src/legacy_registry.rs:143 and delegates/mail-delegate/src/inbox_jobs.rs:84 (procedure only at published-contract/README.md:78-99).
- Remove or rework the v1 send path so the delegate can be built without v1-transition: delegates/mail-delegate/Cargo.toml:62-64, protocol/src/lib.rs:319-324, ui/src/aft.rs:474-484, ui/src/mail_delegate/ops.rs:528-543.
- v1 contacts to v2 ('C2c'): ui/src/address.rs:76-77, ui/src/v2/contacts.rs:181-187.
- Rehearsal against real v0.2.x data: named only at .release-freeze:6; no script or checklist exists.
- Bump the workspace version from 0.2.7: Cargo.toml:56.
- Metadata-disclosure gate in release.sh ('comes with C3'): not present in scripts/release.sh.
- todo!() on node disconnect: ui/src/api.rs:2698. FIXMEs in node error handling: ui/src/api.rs:2590,2597,2670.
- No confirmation tracking after a v1 UPDATE send: ui/src/inbox.rs:689.
- Dead placeholders: ui/src/inbox.rs:545-556 (StoredDecryptedSettings, InboxSummary), ui/src/lib.rs:46 (MAIN_ELEMENT_ID), ui/src/mail_delegate/ops.rs:554-561 (export_profile has no UI).
- Job::SendQueued unimplemented: delegates/mail-delegate/src/jobs.rs:132,153. SealedBody.pass_grant never produced: delegates/mail-delegate/src/inbox_v2.rs:1480,2017.
- v1 inbox never sets last_update (frozen WASM): contracts/inbox/src/lib.rs:1002.
- Ignored AFT integration test awaiting a fixture: modules/antiflood-tokens/contracts/token-allocation-record/tests/test.rs:6-10.
- test.fixme for delete + regenerate identity (#5, #291): ui/tests/live-node.spec.ts:2258,2281.
- Blocked features in the QA inventory: unarchive #60 (docs/qa/manual-test-inventory.md:228), read receipts #69 (:217,263), pad_length and display_name (:264-265), AFT tier configurability #85 (:283), self-heal re-PUT untestable on the iso net (:282).
- Inbox v2 untested anywhere: pass renewal reaching a contact, the real ghostkeys delegate, the vault return link, the strangers-dropped count (docs/qa/manual-test-inventory.md:56-63).
- Loader Phase 2 with rollback to prev_app_ids: contracts/facade-loader/README.md:32-36.
- In-protocol inbox upgrade pointer (#251 improvement 1) and contact key auto-discovery (#42): AGENTS.md only.
- Scrub leaves settings and most read marks in the v0.2.x delegates and never touches token generators: ui/src/migration/scrub.rs:51-54.
- Stray file to delete: modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp. Makefile tasks that build nothing shipped: Makefile.toml:19-226; alias tasks: Makefile.toml:494-513,727,770.
- Docs to rewrite: README.md (whole 'How it works'), RELEASING.md:149-153,198-236, .release-freeze text, published-contract/candidate/README.md:15, the wip-settings sections of docs/qa/manual-test-inventory.md:232-236 and .claude/skills/freenet-mail-qa/SKILL.md:58-83.

### Open questions

- Where are the numbered decisions, 'the plan' (5.3, 5.4, 5.5) and reviews r1-r4 written down (likely upstream PRs #372/#374/#378 or a private document), and what were decisions 7 and 15?
- What is the 'rehearsal against real v0.2.x data': whose data, on which node, and what counts as passing?
- Is v1 sending to be removed in 0.3.0, kept in some reduced form, or is the v1-transition release guard to be relaxed? How do existing v1 contacts become v2 contacts?
- The delegate registries start at v0.2.5 (generation 205). Did v0.2.0-v0.2.4 ship different delegate bytes, and if so are those users' identities unreachable by the migration walk?
- Does anyone besides the developers run main builds against the real network? Those builds PUT candidate inbox v2 contracts under the production Ghost Key master and store data under a candidate delegate key that 0.3.0 will not read.
- Which tracker do the bare #NNN references resolve in today, given README links to iduartgomez/freenet-email and upstream is freenet/mail?
- Would removing .release-freeze really fail the guard 'the freeze override needs the test flag'? Read from the scripts, not run.
- Does the comment at ui/src/app.rs:1976 ('production does not yet fold Sent into the inbox thread') still hold after #310/#311? Not checked in code.
- This fork has no CI runner for the required jobs (ubicloud-standard-8, per docs/codebase-notes/README.md:94-97): where will check-drift and the guard shards run for changes made here?

### Coverage

Read in full: `/home/user/mail-freenet/AGENTS.md (in context)`, `/home/user/mail-freenet/.release-freeze`, `/home/user/mail-freenet/README.md`, `/home/user/mail-freenet/RELEASING.md`, `/home/user/mail-freenet/Cargo.toml`, `/home/user/mail-freenet/.gitignore`, `/home/user/mail-freenet/published-contract/README.md`, `/home/user/mail-freenet/published-contract/candidate/README.md`, `/home/user/mail-freenet/docs/codebase-notes/README.md`, `/home/user/mail-freenet/scripts/check-release-freeze.sh`, `/home/user/mail-freenet/scripts/check-release-delegate.sh`, `/home/user/mail-freenet/scripts/check-release-scrub.sh`, `/home/user/mail-freenet/legacy_inbox.toml`, `/home/user/mail-freenet/legacy_identity_delegate.toml`, `/home/user/mail-freenet/legacy_local_state_delegate.toml`, `/home/user/mail-freenet/ui/branding/app.json`

Skimmed: `/home/user/mail-freenet/Makefile.toml (task list, lines 1-27, 158-232, 356-407, 490-532, 725-776, 820-880, 1262-1292)`, `/home/user/mail-freenet/scripts/release.sh (lines 140-175, 205-236, grep)`, `/home/user/mail-freenet/scripts/release-freeze_test.sh (lines 1-110)`, `/home/user/mail-freenet/scripts/check-guards-fail.sh (lines 70-76, 170-182, 1722-1734, grep)`, `/home/user/mail-freenet/scripts/smoke-test-production.sh (lines 25-40, 85-96)`, `/home/user/mail-freenet/scripts/check-inbox-cost.sh (lines 1-30)`, `/home/user/mail-freenet/scripts/build-wasm.sh (grep)`, `/home/user/mail-freenet/ui/Cargo.toml (lines 30-45, 60-130)`, `/home/user/mail-freenet/ui/build.rs (lines 118-140)`, `/home/user/mail-freenet/ui/src/api.rs (TODO sites, lines 30-46, 178-190, 1262-1280, 2584-2702, 3440-3452, 3576-3586)`, `/home/user/mail-freenet/ui/src/aft.rs (lines 14-58, 277-281, 465-490)`, `/home/user/mail-freenet/ui/src/inbox.rs (lines 44-135, 540-556, 684-694)`, `/home/user/mail-freenet/ui/src/lib.rs (lines 40-52, 104-116, 146-156)`, `/home/user/mail-freenet/ui/src/legacy_registry.rs (lines 1-75, 130-150)`, `/home/user/mail-freenet/ui/src/migration/mod.rs (lines 400-432)`, `/home/user/mail-freenet/ui/src/migration/scrub.rs (lines 1-56)`, `/home/user/mail-freenet/ui/src/mail_delegate/ops.rs (lines 528-560)`, `/home/user/mail-freenet/ui/src/mail_delegate/mod.rs (lines 170-186)`, `/home/user/mail-freenet/ui/src/v2/mod.rs (lines 15-35)`, `/home/user/mail-freenet/ui/src/v2/contacts.rs (lines 176-190)`, `/home/user/mail-freenet/ui/src/v2/send.rs (lines 180-186)`, `/home/user/mail-freenet/ui/src/app/v2_view.rs (lines 97-101, 606-611)`, `/home/user/mail-freenet/ui/src/app/login.rs (lines 222-252)`, `/home/user/mail-freenet/ui/src/app.rs (lines 100-115, 1972-1980)`, `/home/user/mail-freenet/ui/src/address.rs (lines 70-80)`, `/home/user/mail-freenet/ui/tests/live-node.spec.ts (lines 440-452, 545-606, 736-744, 953-965, 2254-2290)`, `/home/user/mail-freenet/ui/tests/email-app.spec.ts (lines 561-580)`, `/home/user/mail-freenet/ui/tests/playwright.config.ts (lines 1-46)`, `/home/user/mail-freenet/ui/tests/production-liveness.spec.ts (grep)`, `/home/user/mail-freenet/ui/tests/app-name.ts (lines 1-16)`, `/home/user/mail-freenet/delegates/mail-delegate/Cargo.toml (lines 50-70)`, `/home/user/mail-freenet/delegates/mail-delegate/src/handlers.rs (lines 1560-1602)`, `/home/user/mail-freenet/delegates/mail-delegate/src/inbox_v2.rs (decision citations only)`, `/home/user/mail-freenet/delegates/mail-delegate/src/inbox_jobs.rs (lines 78-90)`, `/home/user/mail-freenet/delegates/mail-delegate/src/tests/faults.rs (lines 804-812)`, `/home/user/mail-freenet/protocol/src/lib.rs (lines 312-326, 555-565)`, `/home/user/mail-freenet/protocol/src/inbox_v2.rs (lines 25-42, 280-290, 329-333)`, `/home/user/mail-freenet/contracts/inbox-v2/src/merge.rs, types.rs, signed.rs, sync.rs, lib.rs (decision citations only)`, `/home/user/mail-freenet/contracts/inbox-v2/tests/oracle/mod.rs, rules.rs, tamper.rs, metadata_disclosure.rs, plausible_mutants.rs (decision citations only)`, `/home/user/mail-freenet/contracts/inbox-v2/mutants-triage.md (lines 46-60)`, `/home/user/mail-freenet/contracts/inbox/src/lib.rs (lines 1-10, 992-1008)`, `/home/user/mail-freenet/modules/antiflood-tokens/contracts/token-allocation-record/src/lib.rs (lines 160-172, 380-390)`, `/home/user/mail-freenet/modules/antiflood-tokens/contracts/token-allocation-record/tests/test.rs (lines 1-16)`, `/home/user/mail-freenet/modules/antiflood-tokens/delegates/token-generator/tests/integration_test.rs.bkp (first 20 lines)`, `/home/user/mail-freenet/docs/qa/manual-test-inventory.md (lines 36-66, 222-240, 255-268, 282-283, status grep)`, `/home/user/mail-freenet/.claude/skills/freenet-mail-release/SKILL.md (lines 1-60)`, `/home/user/mail-freenet/.claude/skills/freenet-mail-qa/SKILL.md (grep)`, `/home/user/mail-freenet/.github/workflows/e2e-real-node.yml (lines 1-42)`, `/home/user/mail-freenet/contracts/facade-loader/README.md (lines 30-37)`, `/home/user/mail-freenet/tools/web-container-sign/tests/facade_integration.rs (lines 70-90)`, `/home/user/mail-freenet/tools/web-container-sign/src/main.rs (grep for prev_app_ids)`, `sibling maps in the scratchpad maps directory: inbox-v2-contract.md section 11, ui-v2.md lines 9-16, and section 10 of ui-v2, delegate-v2-jobs, protocol-wire, ui-app-shell, delegate-core, build-ci-release, tests-qa`

Skipped: `All other source under ui/src, delegates/mail-delegate/src, contracts/, modules/, protocol/, wire-v1/, common/ and tools/: searched with git grep only, as the task defines this unit as a sweep; their content is covered by the sibling maps`, `.github/workflows/build.yml, check-contract-wasm.yml, e2e-background.yml, inbox-v2-gates.yml, inbox-v2-laws.yml, dependabot.yml: grep hits only (covered by the build-ci-release map)`, `scripts/ other than those listed: grep hits only`, `docs/codebase-notes/*.md area files: this fork's own notes, excluded from every search so they would not count as evidence`, `test-contract/*/README.md, contracts/facade-loader/src/index.html.tmpl, .githooks/: grep hits only`, `Cargo.lock files, ui/tests/package-lock.json, ui/public/, COPYING*, golden and vector .txt fixtures, published-contract binaries: not relevant to the sweep`, `target/ directories: not touched, as instructed`
