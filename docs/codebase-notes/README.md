# Codebase notes

| | |
|---|---|
| Snapshot date | 2026-11-19 |
| Notes version | v2 |
| Snapshot id | `notes-2026-11-19-v2` |
| Describes | commit `2358b59` (upstream `freenet/mail` main of 2026-11-16; this fork had no code changes of its own) |
| Coverage | all 14 areas |

Find a snapshot's commit with `git log --grep=<snapshot id>`. A git tag of the
same name marks it only if the repository owner pushed one: the session that
wrote these notes was allowed to push its branch but not tags.

Reading notes made to plan work on this codebase. They are not project
documentation: `AGENTS.md` stays the authority for rules and procedures, and
where a note and the code disagree, the code is right.

## How far to trust these

- Each area file was written by an AI reader (Claude Code) from the source
  files listed at its end. The readers built and ran nothing.
- Every `path:line` reference is for commit `2358b59`. To see whether an
  area's notes still apply, run
  `git log --oneline 2358b59..HEAD -- <the paths that file covers>`.
- "Suspected bugs and weak spots" are leads from reading, not confirmed
  defects. Only the items under "Checked against the source" below were
  looked at a second time, and none was reproduced by running the app.
- The readers were told to treat `AGENTS.md`, `README.md` and code comments
  as claims to check. Each file lists where those disagree with the code.

## Index

Start with [open-work.md](open-work.md): it says what is unfinished and what
stands between this tree and a release.

| Area | File |
|---|---|
| Cross-cutting: unfinished work, the numbered design decisions, the road to v0.3.0, stale docs | [open-work.md](open-work.md) |
| Inbox v2 contract, its rules and its test gates | [inbox-v2-contract.md](inbox-v2-contract.md) |
| Mail delegate core: request handling, storage, consent, crypto | [delegate-core.md](delegate-core.md) |
| Mail delegate: inbox v2 operations, sealing, background jobs | [delegate-v2-jobs.md](delegate-v2-jobs.md) |
| `protocol/` (page to delegate) and `wire-v1/` (v0.2.x types) | [protocol-wire.md](protocol-wire.md) |
| The frozen v0.2.x generation: v1 inbox, Anti-Flood Tokens, old delegates, webapp and container contracts | [v1-frozen.md](v1-frozen.md) |
| UI core: startup, node WebSocket loop, v1 inbox and Anti-Flood Token clients | [ui-core-v1.md](ui-core-v1.md) |
| UI state: mail delegate client, local mail state, identity and contact stores | [ui-state.md](ui-state.md) |
| UI shell (`ui/src/app.rs`): components, threads, compose and send | [ui-app-shell.md](ui-app-shell.md) |
| UI screens: login, address book, settings, backup, migration status | [ui-app-screens.md](ui-app-screens.md) |
| Inbox v2 in the UI (`ui/src/v2/`, addresses, Ghost Keys) | [ui-v2.md](ui-v2.md) |
| Migration from the v0.2.x delegates, and scrub | [ui-migration.md](ui-migration.md) |
| Build, CI, guards, frozen artifacts, release | [build-ci-release.md](build-ci-release.md) |
| Playwright suites and the QA inventory | [tests-qa.md](tests-qa.md) |

Each area file has the same sections: purpose, walkthrough, interfaces,
invariants and where they are enforced, state, tests, how to change it
safely, disagreements between docs and code, suspected weak spots, open work,
open questions. An appendix holds the reader's summary and the files it read
in full, skimmed and skipped.

## The system in one page

Three layers:

- **Contracts, on the network.** The v1 inbox and the Anti-Flood Token record
  are frozen and are still the mail path of every shipped release. Inbox v2
  is a candidate. The webapp and container contracts are frozen and serve the
  UI.
- **The mail delegate, in the user's node.** It holds every identity's keys,
  the contacts and the local mail state, does all signing, sealing and
  decrypting, and runs the background jobs. It serves only an attested
  web-app origin and keeps each origin's data in its own namespace.
- **The UI, a Dioxus page.** One loop (`api::node_comms`) owns the WebSocket
  and routes every response. The page holds no keys; anything durable is an
  item in the delegate.

v1 and v2 run side by side for the same identity:

- Receiving shows one list: v1 rows, with the drained v2 mail merged in.
- Sending is chosen by the To text alone. A v2 address or a v2 contact's name
  goes v2; everything else goes v1 with a token. There is no fallback either
  way, and a v1 contact cannot yet become a v2 contact.
- v2 has no feature gate. A default build creates a v2 inbox for every
  confirmed identity on load, on the candidate contract.

## Where it stands

The workspace is at 0.2.7, the last release. The v0.3.0 work has landed on
main without a release, and `.release-freeze` blocks one. The freeze file
says scrub, the background jobs and inbox v2 are still to land; all three are
in the tree. What actually stands in the way, in the order
`scripts/release.sh 0.3.0` would fail ([open-work.md](open-work.md) section 5):

1. The freeze file, and the version number.
2. There is no `published-contract/wasm/v2/`. The UI embeds both candidates,
   and the delegate embeds the inbox v2 candidate's code hash.
3. The frozen delegate must be built without `v1-transition`, but v1 sending
   and publishing a new identity both need the operation that feature
   provides (`V1AftParams`). The tree holds no plan for retiring or keeping
   v1 sending. This is the largest piece of work left.
4. The rehearsal against real v0.2.x data that the freeze file names. Nothing
   in the tree records or automates one.

Inbox v2 on real nodes is proven only by hand: `cargo make test-e2e-inbox-v2`
runs in no workflow.

## What any change has to respect

- **Frozen bytes are keys.** A contract or delegate fix after a freeze is a
  new generation plus a data migration.
- **Candidates must match their source.** A change to the delegate,
  `protocol/` or `wire-v1/` means rebuilding the delegate candidate with
  `scripts/build-wasm.sh` (needs `fdev`, linux/amd64). An inbox v2 change
  means rebuilding both, the contract first. Each rebuild changes the code
  hash, so data stored under the previous candidate is not reachable from
  the new build.
- **Guards anchor on exact source text.** About 416 mutations each patch one
  line and expect a named test to fail. Reformatting or renaming an anchored
  line breaks its guard; `scripts/check-guards-fail.sh --anchors` shows it
  without building.
- **Golden files are append-only** (`protocol/src/golden_v1.txt`,
  `wire-v1/src/golden_v1.txt` and four fixture files under `ui/src/`).
- **Stored shapes become permanent at the first release:** the delegate's
  key layout and records, the migration's journal and ledger, the sealed
  envelope, the address and invite text.
- **Two UI feature sets must both pass** (`use-node`, and
  `example-data,no-sync`). Playwright is not part of `cargo make test`.
- **The design record is not in the repository.** The numbered decisions,
  "the plan" and the review rounds that comments cite are defined nowhere in
  the tree. [open-work.md](open-work.md) section 4 reconstructs the decisions
  from their citations.

## The fork at this snapshot

`one137th/mail-freenet` had one branch, `main`, at `2358b59`, with no pull
requests and no issues. It was not compared against `freenet/mail` after
that commit.

CI as written will not run here: every heavy job (build and test, the
required `check-drift`, both real-node suites) has
`runs-on: ubicloud-standard-8`, a third-party runner. Only the inbox v2 gates
and the nightly merge laws use GitHub-hosted runners.

## Leads worth knowing

From reading; unconfirmed unless also listed in the next section.

- The node loop has no reconnect, and a dropped socket ends it
  ([ui-core-v1.md](ui-core-v1.md) section 9).
- A v1 send in progress lives only in page memory: closing the tab loses it,
  and several paths leave a Sent row stuck or wrongly failed. This is the
  code behind the symptoms of upstream issue #320
  ([ui-core-v1.md](ui-core-v1.md) sections 3 and 9).
- A v2 send is not resumed after the tab closes, and a reload can mark a
  waiting one failed ([ui-v2.md](ui-v2.md) section 9).
- The quarantine order in inbox v2 can be ground by a sender. The contract
  is still a candidate, so this can be changed before the freeze
  ([inbox-v2-contract.md](inbox-v2-contract.md) section 9).
- The v1 inbox does not bind a token to its record. That code is frozen
  ([v1-frozen.md](v1-frozen.md) section 9).
- An identity's inbox v2 record is one item capped at 256 KiB and may be
  able to outgrow it ([delegate-v2-jobs.md](delegate-v2-jobs.md) section 9).
- A cap reached during migration is a permanent rejection with no retry
  path, which then blocks scrub for good
  ([ui-migration.md](ui-migration.md) section 9).
- `README.md` describes the v0.2.x architecture, and the QA inventory and
  the release skill are stale ([open-work.md](open-work.md) section 8,
  [tests-qa.md](tests-qa.md) section 8).

## Checked against the source

Looked at a second time on 2026-11-19, at `2358b59`. Each is a reading of
the code; none was reproduced by running the app.

- Runner labels: `ubicloud-standard-8` at `build.yml:30,168`,
  `check-contract-wasm.yml:160,411`, `e2e-background.yml:45`,
  `e2e-real-node.yml:43`. `ubuntu-latest` at `check-contract-wasm.yml:62,513,539`
  and throughout `inbox-v2-gates.yml` and `inbox-v2-laws.yml`.
- Candidate code hashes: mail delegate
  `Bavf78hk722usjefjPAkrbjtyaJwJbRpgq6nFVqvp2vF`, inbox v2
  `GpuWR879xBi13sSeDhoT1zNzEhVmMBFSEr1aoJGCc8a6`. `published-contract/wasm/`
  holds only `v1`.
- The delegate's origin gate serves only `MessageOrigin::WebApp` with a
  32-byte id that is not all zero; another delegate or an unattested caller
  gets a `DelegateError` (`delegates/mail-delegate/src/handlers.rs:123-147`).
- With `v1-transition`, `V1AftParams` returns parameters built from the
  identity's ML-DSA seed with no prompt; without it the reply is `Refused`
  (`handlers.rs:1587-1600`).
- The node loop unwraps the WebSocket send (`ui/src/api.rs:3768`) and its
  `RequestError::Disconnect` arm is `todo!()` (`api.rs:2697-2699`).
- The handler for a failed contract UPDATE acts only when the key is one of
  the user's own token records or is in `INBOX_TO_ID`
  (`api.rs:2595-2632`, `ui/src/inbox.rs:1244-1246`). The reader's conclusion
  that a failed UPDATE to a contact's inbox is therefore dropped rests on
  `INBOX_TO_ID` holding own inboxes only; the callers that fill it were
  listed, not read.
- `reseal_at` is `sent_time + 2 d - (1 d + skew + 6 h)`, never before
  `now + 6 h`: 18 hours after `sent_time`, not the two days `AGENTS.md`
  gives (`delegates/mail-delegate/src/inbox_v2.rs:2090-2094`, constants at
  `:58`, `:85` and `protocol/src/inbox_v2.rs:34`).
- A v2 inbox is started for every confirmed identity at load, under no
  feature check (`ui/src/app/identity_store.rs:847-851`).
- Compose refuses an invite, then sends v2 when
  `v2::contacts::recipient_addr` resolves the To text, else v1
  (`ui/src/app.rs:6175-6186`).
- `Job::SendQueued` appears only in the planner
  (`delegates/mail-delegate/src/jobs.rs:132,153`); nothing runs it.
- Adding a v2 contact refuses a v0.2 address with "Old contacts move over in
  a later update" (`ui/src/v2/contacts.rs:181-187`).
- With threading off the list groups with `group_flat`
  (`ui/src/app.rs:4091-4097`) while the detail pane always resolves with
  `group_into_threads` (`app.rs:4217-4219`). The consequence the reader
  draws, that a reply row cannot be opened, was not run.
- Settings, Account, "Restore…" closes Settings and sets `ImportBackup`
  (`ui/src/app/settings.rs:465-468`); `ImportForm` is rendered only in
  `ui/src/app/login.rs` (`:936`, `:1842`). The consequence, that nothing
  appears while an identity is open, was not run.

## Test baseline

Run on 2026-11-19 at `2358b59`, Linux x86_64, 4 CPUs, rustc 1.99.1. All
passed.

| Command | Result | Time |
|---|---|---|
| `cargo test --workspace` | UI 591, protocol 17, wire-v1 71, web-container-sign 6 + 2 | 210 s |
| `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` | 636 | 67 s |
| `scripts/check-excluded-crates.sh test` | mail delegate 279 in each feature set; inbox v2 106, and 126 with `plausible-mutants`; web-container 6; AFT interfaces 19, token record 3 (1 ignored), token generator 7; identity-management 16; mail-local-state 31 | 557 s |

These three are what `cargo make test` chains. Not run: `fmt`, `clippy`,
the UI build, `cargo make test-inbox` (the v1 inbox contract's tests need
`--features contract`), Playwright, the real-node suites, the guards, the
candidate rebuild checks.

## Tools

Present where these notes were made: rustup with 1.99.1 and the wasm32
target, Node 22, Python 3.13, GNU tar, `flock`, `perl`, `openssl`, Playwright
with Chromium. Missing, and needed for the steps named: `cargo-make` (every
`cargo make` task), `dx` 0.7.10 (UI builds, Playwright suites), `fdev` and
`freenet` (candidate rebuilds, frozen-artifact checks, real-node suites; CI
pins freenet-core v0.2.151), `b3sum` and Python `cbor2` (the vector check),
`wasmtime` (the cost gate), `xxd`, `cargo-llvm-cov`, `cargo-mutants`.

## Corrections and known disagreements between the files

- `build-ci-release.md` had the two candidate code hashes swapped in its
  "State" section. Fixed in the file.
- `open-work.md` and `ui-app-screens.md` were written while v1 of these
  notes was already in the tree, and mention it.
- The files give different current locations for the two contact priming
  GETs that `AGENTS.md` cites as "api.rs ~1442 and ~2540". They are
  `prime_contact_inbox` at `ui/src/api.rs:420`, called from `:1836` and
  `:1920`, and the `CreateContact` arm near `:2282`.

## Related

Product-level research that preceded these notes:
`docs/freenet-email-research.md` in `one137th/Email` (2026-11-14, read at
upstream `addb776`).

## Version history

| Version | Date | Snapshot id | Contents |
|---|---|---|---|
| v1 | 2026-11-19 | `notes-2026-11-19-v1` | 6 of 14 areas, the test baseline, the first checks |
| v2 | 2026-11-19 | `notes-2026-11-19-v2` | all 14 areas, the cross-cutting summary, more checks |
