# Codebase notes

| | |
|---|---|
| Snapshot date | 2026-11-19 |
| Notes version | v1 |
| Snapshot id | `notes-2026-11-19-v1` |
| Describes | commit `2358b59` (upstream `freenet/mail` main of 2026-11-16; this fork had no changes of its own) |
| Coverage | 6 of 14 areas. The other 8 were still being read when this was saved and come in v2. |

Find a snapshot's commit with `git log --grep=<snapshot id>`. A git tag of the
same name marks it only if the repository owner pushed one: the session that
wrote v1 was allowed to push its branch but not tags.

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
  looked at a second time.
- The readers were told to treat `AGENTS.md`, `README.md` and code comments
  as claims to check. Each file lists where those disagree with the code.

## Index

| Area | File | In v1 |
|---|---|---|
| Mail delegate core: request handling, storage, consent, crypto | [delegate-core.md](delegate-core.md) | yes |
| Inbox v2 contract, its rules and its test gates | [inbox-v2-contract.md](inbox-v2-contract.md) | yes |
| UI core: startup, node WebSocket loop, v1 inbox and Anti-Flood Token clients | [ui-core-v1.md](ui-core-v1.md) | yes |
| UI state: mail delegate client, local mail state, identity and contact stores | [ui-state.md](ui-state.md) | yes |
| Build, CI, guards, frozen artifacts, release | [build-ci-release.md](build-ci-release.md) | yes |
| Playwright suites and the QA inventory | [tests-qa.md](tests-qa.md) | yes |
| Mail delegate: inbox v2 operations, sealing, background jobs | delegate-v2-jobs.md | pending |
| `protocol/` and `wire-v1/` | protocol-wire.md | pending |
| Inbox v2 in the UI (`ui/src/v2/`, addresses, Ghost Keys) | ui-v2.md | pending |
| UI shell (`ui/src/app.rs`): components, threads, compose and send | ui-app-shell.md | pending |
| Migration from the v0.2.x delegates, and scrub | ui-migration.md | pending |
| UI screens: login, address book, settings, backup | ui-app-screens.md | pending |
| The frozen v0.2.x generation: v1 inbox, Anti-Flood Tokens, old delegates, webapp and container contracts | v1-frozen.md | pending |
| Cross-cutting sweep: TODOs, the numbered design decisions, the road to v0.3.0, stale docs | open-work.md | pending |

Each area file has the same sections: purpose, walkthrough, interfaces,
invariants and where they are enforced, state, tests, how to change it
safely, disagreements between docs and code, suspected weak spots, open work,
open questions. An appendix holds the reader's summary and the files it read
in full, skimmed and skipped.

## The project at this snapshot

- **Between releases.** The workspace is at 0.2.7, the last release. Since
  then the v0.3.0 work has landed on main without a release: frozen artifacts
  (#353), the unified mail delegate and its protocol (#364), the UI switched
  to it (#366), migration from the v0.2.x delegates (#368) and scrub (#369),
  background jobs (#370), the inbox v2 contract (#372), its delegate
  operations (#374) and its UI (#378).
- **Releases are frozen** by `.release-freeze`. Its text says scrub, the
  background jobs and inbox v2 are still to land; all three are in the tree.
  What the tree still lacks for 0.3.0: a frozen `published-contract/wasm/v2/`
  (the UI embeds the unfrozen candidates of the mail delegate and of inbox
  v2), and a v1 send path that does not need the delegate's `v1-transition`
  feature, which `scripts/check-release-delegate.sh` refuses from 0.3.0.
  The freeze file also names a rehearsal against real v0.2.x data.
- **Three kinds of artifact** ([build-ci-release.md](build-ci-release.md)):
  frozen (never rebuilt), candidates (rebuilt by the contributor on every
  source change, byte-compared in CI), and per-release signed state.
- **A change to the delegate, `protocol/`, `wire-v1/` or the inbox v2
  contract** means rebuilding a candidate with `scripts/build-wasm.sh`
  (needs `fdev`, linux/amd64) and runs every guard shard in CI. An inbox v2
  change means rebuilding both candidates, because the delegate embeds the
  contract's code hash (`delegates/mail-delegate/src/inbox_jobs.rs:83-85`).
- **Guards anchor on exact source text.** Reformatting or renaming an
  anchored line breaks its guard; `scripts/check-guards-fail.sh --anchors`
  shows it without building.
- **The numbered design decisions** the code cites ("decision 9", "decision
  25") are defined nowhere in the tree.
  [inbox-v2-contract.md](inbox-v2-contract.md) section 11 reconstructs them
  from the comments that cite them.

## The fork at this snapshot

`one137th/mail-freenet` had one branch, `main`, at `2358b59`, with no pull
requests and no issues. It was not compared against `freenet/mail` after
that commit.

CI as written will not run here: every heavy job (build and test, the
required `check-drift`, both real-node suites) has
`runs-on: ubicloud-standard-8`, a third-party runner. Only the inbox v2 gates
and the nightly merge laws use GitHub-hosted runners.

## Checked against the source

Looked at a second time on 2026-11-19, at `2358b59`:

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

## Corrections made to reader output

- `build-ci-release.md` had the two candidate code hashes swapped in its
  "State" section. Fixed here; the appendix was not affected.

## Related

Product-level research that preceded these notes:
`docs/freenet-email-research.md` in `one137th/Email` (2026-11-14, read at
upstream `addb776`).

## Version history

| Version | Date | Snapshot id | Contents |
|---|---|---|---|
| v1 | 2026-11-19 | `notes-2026-11-19-v1` | 6 of 14 areas, the test baseline, the checks above |
