> Notes v1, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Map: build, CI, guards, frozen artifacts, release (`build-ci-release`)

Read at 2358b59 (branch `claude/kind-einstein-k282vt`, `origin` = one137th/mail-freenet). Nothing was built or run: every "does" below is from reading the file. Paths are repo-relative.

## 1. Purpose and where it sits

This subsystem decides which bytes ship and proves they are the reviewed ones. It handles three classes of artifact:

- **Frozen**: `published-contract/{web_container_contract,container}.wasm`, their parameters and ids, and `wasm/v1/*` (18 sha256 rows in `published-contract/frozen.sha256`). Never rebuilt. The UI embeds v1 with `include_bytes!` (`ui/src/api.rs:590,768,1037,1182,1317`).
- **Candidates**: `published-contract/candidate/{mail_delegate,inbox_v2}` plus `*_code_hash`. The contributor rebuilds them on every source change; CI rebuilds and `cmp`s. The UI embeds both today (`ui/src/mail_delegate/node.rs:19-22`, `ui/src/v2/mod.rs:27-29`).
- **Per-release state**: the signed UI archive and the container pointer, made by `scripts/release.sh` with an uncommitted production key. Blocked now by `.release-freeze`.

The guard machinery (`check-guards-fail*.sh`, `select-guards.sh`, shards) exists to show each check failing when what it guards is broken.

## 2. Walkthrough

### Root config
- `Cargo.toml`: members L3-18 (common, protocol, ui, wire-v1, tools/web-container-sign, tools/v1-node-seed); `exclude` L31-53 (every contract and delegate, each with its own `Cargo.lock`); version `0.2.7` L56; `freenet-stdlib 0.12.1` L62; release profile L102-107; `wasm-release` (strip=debuginfo, used by dx) L113-115.
- `rust-toolchain.toml:12` pins `1.99.1` with wasm32, rustfmt, clippy. `contracts/facade/rust-toolchain.toml` pins `1.95.0`; `scripts/container-toolchain.sh:12-29` forces it with `RUSTUP_TOOLCHAIN` and fails unless `rustc`/`cargo` report it (cargo-make exports the root toolchain).
- `.gitignore`: `target/`, `**/build/` (any directory named `build`), `dist/`.
- `.claude/settings.json`: enables the plugins `playwright@claude-plugins-official` and `freenet@freenet-agent-skills` (marketplace github `freenet/freenet-agent-skills`). No permissions or hooks.
- `.githooks/pre-commit` (not enabled in this checkout: `core.hooksPath` is unset): blocks M/D/T of any path in HEAD's `frozen.sha256` or under `published-contract/wasm/` (L38-43), blocks removed or changed manifest rows (L45-49), blocks any staged `*.wasm` except the two top-level ones (L22, L52-66). Candidate files have no extension and are not frozen, so the hook lets them change.

### `Makefile.toml` (cargo-make, `skip_core_tasks`)
- fdev rebuilds into each crate's `build/` (L28-125, `build-contracts` L207): never embedded or published.
- UI: `build-ui` L515-527 deletes `target/dx/app/release/web/public`, then `dx build --release --features "use-node${FREENET_UI_FEATURES:+,…}"`. `build` = `build-ui` L529. `dev` L533-545 refuses if `FREENET_MAIL_SCRUB` is set. `build-ui-example-no-sync` L554.
- Publish chain: `compress-webapp` L590 (GNU tar, `--sort=name --mtime='2024-01-01'`, output `target/webapp/webapp.tar.xz`) → `sign-webapp-test` L636 (runs `check-release-scrub.sh --absent` unless `FREENET_MAIL_SCRUB_ISO=1`, version = `date -u +%s`) or `sign-webapp` L675 (production key, `WEB_CONTAINER_KEY_FILE`) → `derive-test-contract-id` L706 (`fdev get-contract-id`, writes `target/webapp/contract-id.txt`) or `verify-published-contract` L734 (cmp parameters; id pinned at L757) → `publish-email-test` L777 (port `FREENET_LOCAL_PORT:-7509`, strips the probe and ghost opt-ins before the scrub check L787), `publish-email-iso` L803 (runs `run-isolated-nodes.sh check-ours` first), `publish-email` L837 (deps: freeze, sign, `check-webapp-scrub`, verify).
- `publish-production` L861 runs `FREENET_MAIL_SCRUB=1 cargo make publish-production-flow` (L872: freeze, `build`, `publish-email`).
- Container: `build-container-loader` L234, `build-container` L252, `pack-container-loader` L267 (greps the app id into the HTML, tar.xz), `sign-container-state{,-test}` L316/L358, `update-published-container` L389 (sandbox id only), `publish-container{,-test}` L408/L436, `update-container-state-test` L475. Facade-named aliases L493-513.
- Tests: `test` L907 = `test-workspace` + `test-ui-offline` (L916: `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync`) + `test-excluded` (`scripts/check-excluded-crates.sh test`). `test-inbox` L926. `clippy` L935. `check-golden-capture` L949 (diffs three fixtures against `wire-v1/capture` output).
- Playwright: `test-ui-playwright` L971 (free port 8082..8182, `lsof`, `curl`).
- Real-node tasks: `test-e2e-real-node` L1068, `test-e2e-inbox-v2` L1133 (wrapper sets `FREENET_TEST_GHOST_ISO=1 FREENET_UI_FEATURES=test-ghost-master`), `test-e2e-background` L1198 (`FREENET_E2E_PROBE_ISO=1 FREENET_UI_FEATURES=e2e-probe`), `test-e2e-legacy-migration` L1289 (`FREENET_MAIL_SCRUB=1 FREENET_MAIL_SCRUB_ISO=1`), `capture-v0-2-7-replies` L1265. Each picks free ports with python3 and asserts that `check-release-scrub.sh` refuses or accepts its build as expected (L1107-1114, L1177-1184, L1241-1248, L1327).

### Candidate and frozen scripts
- `scripts/build-wasm.sh <crate> <contract|delegate> <features> <out>`: copies the work tree (tracked plus untracked-not-ignored files, as on disk) to the fixed path `/tmp/freenet-mail-wasm-build` under `flock` (L61-78); remaps `CARGO_HOME→/cargo`, `RUSTUP_HOME→/rustup`, stage→`/build`, target→`/target` (L85-90); re-adds `--cfg getrandom_backend="custom"` because env rustflags replace the `.cargo` ones (L91-96); runs `fdev build --features … [--package-type delegate]` (L99-102); refuses output containing a machine path (L108, `check-wasm-paths.sh:21-33`); prints `code hash:` from `fdev inspect` (L113). Needs fdev, rustup with `~/.rustup` or `RUSTUP_HOME` present (L50 fails otherwise), `flock`, GNU tar, a writable `/tmp`.
- `scripts/check-delegate-candidate.sh:24-45`: rebuilds with features `freenet-main-delegate,v1-transition`, `cmp`s against `candidate/mail_delegate`, compares `mail_delegate_code_hash`; on mismatch keeps the rebuild in `target/delegate-candidate-rebuilt/` (CI uploads it).
- `scripts/check-inbox-candidate.sh:23-56`: the same for `contracts/inbox-v2` (kind contract, feature `contract`), and fails if the WASM names `__frnt__time__utc_now` or `__frnt__rand__rand_bytes` (L34-39).
- `scripts/check-delegate-imports.sh`: a python WASM import-section parser, checked against `scripts/delegate-imports-allowlist.txt` (17 rows). Default input is the delegate candidate; CI also runs it on `inbox_v2`.
- `scripts/check-frozen-artifacts.sh`: byte pins L66-84; per-generation known-file and code-hash checks L96-140 (five artifact names hard-coded L43-49, v1 hashes pinned); contract ids L160-165 (pinned L39-40). Needs fdev.
- `scripts/check-frozen-immutable.sh <base>`: no M/D/T under `published-contract/wasm/` (L69); manifest rows append-only (L76-85); `legacy_*.toml` rows append-only (L90-116); six fixture files append-only by line (L123-144: `wire-v1/src/golden_v1.txt`, `ui/src/frozen_keys_v1.txt`, `ui/src/inbox/{crypto,kem}_vectors_v1.txt`, `ui/src/register_defaults_v1.txt`, `protocol/src/golden_v1.txt`).
- `scripts/check-wasm-builds.sh`: builds the five v1 sources through `build-wasm.sh` and web-container with cargo; output discarded.
- `scripts/check-container-byte-equal.sh`: rebuilds `contracts/facade` with 1.95.0 and `cmp`s; skips off linux/x86_64 (L35-39). `scripts/build-container-snapshot-linux.sh` regenerates the snapshot (a deliberate re-key).
- `scripts/check-excluded-crates.sh fmt|clippy|test`: ten crates (L12-23), `--locked`, extra pass with `v1-transition` (delegate) and `plausible-mutants` (inbox-v2) L28-34.

### Guard system
- `scripts/check-guards-fail.sh` (1780 lines) sources `check-guards-fail-jobs.sh` and `check-guards-fail-inbox.sh` (L1640-1641). By grep: 416 mutation guards (309 + 91 + 16) plus one command guard ("release guard", L1741-1762); `--no-wasm` drops that one and three more (L1729-1738).
- Mechanism: stage the tree with `git ls-files` into `target/guards.XXXXXX` (L70-82; needs a git work tree); run 24 baselines (L207-230); per guard `replace` an exact string that must occur exactly once (L237-246), run the guard's command, require the named `test … FAILED` lines or a fixed message (L251-262, L272-318). Shared target dirs `target/guards-cargo` and `target/guards-cargo-delegate` (L73, L96).
- Flags: `--list`, `--list-files` (name TAB mutated file), `--list-cmds`, `--only FILE`, `--shard I/N` (guard index mod N; under `--only` the index counts selected guards, L63-68), `--anchors` (only checks every mutation still applies; builds nothing, L286-296), `--no-wasm`. Env `GUARD_ONLY=<regex>` runs a subset (L282). A shard that ran nothing fails (L1766-1769).
- Mutated files (count of guards): `delegates/mail-delegate/src/{inbox_v2,handlers,watch,jobs,inbox_jobs,store,consent,sealing,env}.rs`, `ui/src/migration/{scrub,legacy_delegates,successor,mod,orphans,node}.rs`, `ui/src/v2/{owner,net,contacts,queue}.rs`, `ui/src/{app,api,jobs,ghostkeys}.rs`, `ui/src/app/{identity_store,migration_view,settings}.rs`, `ui/src/local_state/items.rs`, `protocol/src/{lib,address,inbox_v2}.rs`, `contracts/inbox-v2/src/{merge,types,signed}.rs`, its `Cargo.toml`, `tests/metadata_disclosure.rs`, `tests/fixtures/vectors.txt`, `docs/metadata-fields.toml`, `Makefile.toml` (8), `scripts/{release,check-release-scrub,check-release-freeze,check-mutants-triage,check-fdev-verify-merge}.sh`, `check-branch-coverage.py`, `delegate-imports-allowlist.txt`.
- `scripts/select-guards.sh EVENT < paths`: every guard unless the event is `pull_request` and no `HARNESS` path changed (L26: `scripts/`, `published-contract/`, `.github/`, `.githooks/`, `.release-freeze`, `Cargo.lock`, `rust-toolchain.toml`, `Makefile.toml`, any `.cargo/`, any `Cargo.toml`). Otherwise by area: `area_inputs` L27-35 (delegate, inbox, protocol, ui, harness), `area_of` L37-46 maps a guard's mutated file to its area and exits 1 for an unmapped file. An area hit with an empty selection exits 1 (L73-76).
- `scripts/select-guards_test.sh`: asserts selections for sample diffs (L29-80), that each area's regex covers the crate's path-dependency closure via `cargo metadata --offline` (L83-120), a scan of every tracked `.rs` for `include*!`, `fs::` and `File::open` whose path must resolve to an area input (L122-348; an unresolvable read is refused unless in `EXEMPT` L147-154), that each guard is selected by its own mutated file (L350-358), and that 8 shards partition a selection (L366-375).
- `scripts/check-guard-shards.sh`: parses `check-contract-wasm.yml`: the matrix must be exactly `0..N-1` (L11-16), the step must say `--shard ${{ matrix.shard }}/N` (L17), `check-drift` must need `guard-shards` (L21), the nightly skip must be schedule-only and `NIGHTLY_REPEAT` must appear exactly twice (L26-30); the shards' union must equal the full list (L33-50).

### Gates on inbox v2 (slow)
`check-branch-coverage.sh` + `.py` (cargo-llvm-cov `--branch` on a nightly; allowlist `contracts/inbox-v2/coverage-allowlist.toml`; needs python ≥ 3.11 for `tomllib`), `check-mutants-triage.sh` (`--complete`, `--min-tested` default 10), `check-fdev-verify-merge.sh` (strips the 40-byte fdev header L47-51, wants ≥ 12 states, 200 cases, no inconclusive), `check-inbox-cost.sh` (wasmtime fuel budget 37e9 L42; expected return values per case L65-72), `check-inbox-vectors.sh` (`$VECTORS_PYTHON -I vectors.py`, needs cbor2 and b3sum).

### Release scripts
- `check-release-freeze.sh`: exit 3 while `.release-freeze` exists; `RELEASE_FREEZE_FILE` is honoured only with `RELEASE_FREEZE_TEST=1` (L18-25).
- `check-release-scrub.sh [--absent] [--dir]`: reads the single `*.wasm` under `target/dx/app/release/web/public` for three markers: scrub (L32), `e2e-probe` (L39), `test-ghost-master` (L45). Refuses zero or more than one WASM file (L74-83).
- `check-release-delegate.sh <version>`: passes below 0.3.0 (L66-69); otherwise refuses any `published-contract/candidate` reference under `ui/` (L72-77), a missing `wasm/v2/mail_delegate` (L78), the `v1-transition` marker (L82), and a byte mismatch with a fresh `build-wasm.sh` build using only `freenet-main-delegate` (L90-96).
- `release.sh <version> [--yes]`: freeze gate first (L37); preflight L88-211 (branch `main`, clean tree, tag free, `HEAD == origin/main`, workspace version equals the argument, delegate check, key file, fdev/dx/cargo-make, `fdev execute update --as-state` support, GNU tar, node on `FREENET_PORT:-7509`, frozen artifacts); `cargo make test`, `test-inbox`, `clippy` (L216-218); `test-e2e-real-node` unless `FREENET_RELEASE_E2E=0` (L226); `cargo make publish-production` (L285); refuse if `published-contract/` changed (L291); container flip (L312-344); `release-tag.sh --sha` (L354).
- `release-tag.sh`: the `--sha` path (L94-96) is the one in use; the snapshot-commit path (L97-131, L165-173) belongs to a superseded flow.
- `generate-production-key.sh`, `smoke-test-production.sh` (runs `ui/tests/production-liveness.spec.ts`), `build-loader.sh` (base58 43-44 char check L25, placeholder checks L43, L56).
- `run-isolated-nodes.sh up|down|wipe|status|check-ours`: two `freenet network` nodes under `FREENET_E2E_ROOT` (`target/iso` locally, `~/freenet-mail-iso` under `CI`, L30-34); refuses a listener its pidfile does not name (L148-178); blocks auto-update with a dead proxy (L77). Needs freenet, perl, openssl, xxd, curl.

### `tools/web-container-sign`
A workspace member. Subcommands `generate`, `sign` (payload `version u32 BE || archive`, output CBOR `WebContainerMetadata` plus 32-byte parameters, `src/main.rs:326-384`), `show-pub`, `sign-facade-state` (`FacadePointer`, state `[meta_len u64 BE][CBOR meta][web_len u64 BE][web]`, L166-270). Key file: TOML `[keys] signing_key, verifying_key` in base58 (L120-132), written 0600. Types come from `freenet_email_core::{facade, web_container}` (L35-38). Tests: six unit tests (L442-604) and `tests/facade_integration.rs`, which runs the binary and feeds its output to `FacadeContract::validate_state` (L17-69); the second test there asserts nothing (L76-88).

### `contracts/facade-loader`
`src/index.html.tmpl`: `__CURRENT_APP_ID__` appears twice (L24, L31); when framed it posts `{__freenet_shell__, type:'navigate', href}` to the parent (L60-65), else `location.replace` (L69). `dist/` is ignored by git.

### Workflows
| Workflow | Triggers | Jobs → runner |
|---|---|---|
| `build.yml` | every PR; push main | `build-and-test`, `ui-playwright-tests` → `ubicloud-standard-8` (L30, L168) |
| `check-contract-wasm.yml` | every PR; nightly 05:20; dispatch; push main (path filter L20-46) | `changes`, `check-drift`, `report-nightly-failure` → `ubuntu-latest`; `drift-checks`, `guard-shards` ×8 → `ubicloud-standard-8` (L160, L411) |
| `e2e-real-node.yml` | every PR (L18); tags `v*`; nightly 04:30; dispatch | `ubicloud-standard-8` (L43) |
| `e2e-background.yml` | PR paths L13-22; nightly 05:30; dispatch | `ubicloud-standard-8` (L45) |
| `inbox-v2-gates.yml` | PR paths L15-24; nightly 05:15; dispatch | five jobs, all `ubuntu-latest` (`mutants` is ×8, 120 min each) |
| `inbox-v2-laws.yml` | nightly 04:45; dispatch | `ubuntu-latest` |

- `build-and-test` steps (`build.yml:104-165`): `cargo fmt --all`, `cargo make build`, `clippy`, `test`, a release fault sweep (`FAULT_SEEDS=2000 FAULT_OPS=80 FAULT_RATE=200 FAULT_BLIND_RATE=20`, L120-129), `test-inbox`, the facade crate (fmt, clippy, test, wasm build), `check-excluded-crates.sh fmt`, `check-golden-capture`, `release-freeze_test.sh`, `check-release-scrub_test.sh`.
- `drift-checks` steps (`check-contract-wasm.yml:206-401`): `select-guards_test.sh`, parameters tripwire (hex at L214 and L388, uses `xxd`), frozen artifacts, frozen immutable, `check-wasm-builds.sh`, both candidate checks, imports on both candidates, `check-inbox-vectors.sh` (cbor2 5.8.0, b3sum 1.8.5), disclosure gate and plausible mutants, `check-guard-shards.sh`, `check-guards-fail.sh --anchors`, container byte-equality.
- `changes` (L61-148): docs-only detection (L87-96: only `*.md`, LICENSE, COPYING, and nothing under `ui/public/`), nightly-repeat lookup with `gh run list` (L118-139), guard selection.
- `check-drift` (L510-528) succeeds only if `changes`, `drift-checks` and `guard-shards` all report `success`; it is skipped for docs-only PRs.
- Tool pins: fdev and freenet from freenet-core release `v0.2.151` (fdev 0.3.313), dioxus-cli 0.7.10, Node 20, `nightly-2025-08-08`, cargo-llvm-cov 0.6.21, cargo-mutants 27.1.0 (`inbox-v2-gates.yml:33-36`).
- No `secrets.*` reference in any workflow; only `github.token` (L122, L547).
- `dependabot.yml`: cargo at `/` and github-actions, monthly, 7-day cooldown, grouped minor and patch; ignores `dtolnay/rust-toolchain` (L98). Excluded crates are deliberately not listed.

## 3. Interfaces

- **Build-time env read by the UI**: `FREENET_MAIL_SCRUB` (`ui/src/migration/scrub.rs:83`, `option_env!`), `FREENET_UI_FEATURES` (Makefile L526 only), `FREENET_EMAIL_FACADE_ID` (set by `ui/build.rs:49` from `container-id.txt`). `ui/build.rs:51-92` requires the twelve v1 files and `contract-id.txt` for a `use-node` build.
- **Markers greppable in the built WASM**: `freenet-mail:scrub-built-in`, `freenet-mail:e2e-probe-built-in`, `freenet-mail:test-ghost-master-built-in` (`check-release-scrub.sh:32-45`); `freenet-mail-delegate:v1-transition-enabled` in the delegate (`check-release-delegate.sh:41`).
- **Candidate consumers**: UI (above); `ui/src/legacy_registry.rs:143` (delegate code hash); the delegate itself embeds `candidate/inbox_v2_code_hash` (`delegates/mail-delegate/src/inbox_jobs.rs:83-85`) and the v1 inbox hash (`handlers.rs:1466-1468`); a delegate test reads the committed delegate candidate (`delegates/mail-delegate/src/tests/jobs.rs:173-177`).
- **fdev**: `build`, `inspect <file> code|delegate`, `get-contract-id`, `publish … contract`, `execute update --as-state`, `verify-merge`.
- **Opt-in env for harness publishes**: `FREENET_MAIL_SCRUB_ISO`, `FREENET_E2E_PROBE_ISO`, `FREENET_TEST_GHOST_ISO`; ports `FREENET_ISO_{GW,PEER}{,_NET}_PORT`, `FREENET_LOCAL_PORT`, `FREENET_PORT`.

## 4. Invariants and where each is enforced

| Rule | Enforced by | Shown failing by |
|---|---|---|
| Frozen bytes and pin rows never change | `check-frozen-immutable.sh`; pre-commit hook | no guard (history check) |
| Frozen tree is self-consistent, ids pinned | `check-frozen-artifacts.sh:39-49` | no guard |
| Candidate equals the source's build | the two candidate checks | guards at `check-guards-fail.sh:1730-1738` |
| Contract build cannot carry the test seam | `contracts/inbox-v2/src/lib.rs` compile error | guard L1736 |
| Delegate imports are within the host set | `check-delegate-imports.sh` | `check-delegate-imports_test.sh`; guard `check-guards-fail-jobs.sh:12` |
| Container rebuild equals the snapshot | `check-container-byte-equal.sh` | no guard |
| No production publish under freeze | `check-release-freeze.sh`; Makefile deps | `release-freeze_test.sh`; guards L1648-1655, L1726 |
| Production build has scrub; test publish has none; no probe or test master ships | `check-release-scrub.sh` | `check-release-scrub_test.sh`; guards L1656-1725 |
| v0.3.0+ ships a frozen delegate built without `v1-transition` | `check-release-delegate.sh` | `check-release-delegate_test.sh` |
| Every guard runs exactly once across shards | `check-guard-shards.sh` | none |
| Every mutation still applies | `--anchors` in `drift-checks` | n/a |
| A guarded change never selects zero guards | `select-guards.sh:73-76` | `select-guards_test.sh` |

## 5. State

- **Shipped and frozen**: webapp contract `GeQLXXR8…`, container contract `122f6AR7…`, both with the same production key parameters (two identical sha256 rows in `frozen.sha256`), and v1 (five WASM, five hashes, two params).
- **Candidate**: `mail_delegate` (code hash `Bavf78hk…`, built with `v1-transition`), `inbox_v2` (`GpuWR879…`). Both contain only `/cargo/…` and `/build/…` paths (checked with `strings`).
- **Unfinished**: `.release-freeze` is present; there is no `published-contract/wasm/v2/`; workspace version is 0.2.7 while the delegate crate is 0.3.0.

## 6. Tests and how to run them

| Command | Proves | Needs |
|---|---|---|
| `bash scripts/select-guards_test.sh` | guard selection, read scan | cargo (offline metadata), python3, git |
| `bash scripts/check-guard-shards.sh` | shard coverage, workflow shape | bash ≥ 4 |
| `bash scripts/check-guards-fail.sh --anchors` | every mutation applies once | python3, git, GNU tar |
| `scripts/check-guards-fail.sh [--no-wasm]` | each guard fails when broken | cargo 1.99.1, fdev, flock, cbor2, b3sum; hours on one machine (CI: 8 shards, 90 min limit each, 8 cores) |
| `scripts/release-freeze_test.sh`, `check-release-scrub_test.sh` | freeze and scrub gates | bash, awk |
| `scripts/check-delegate-imports_test.sh` | import check | python3 |
| `scripts/check-release-delegate_test.sh` | release guard on real builds | fdev, flock |
| `scripts/check-branch-coverage_test.sh`, `check-mutants-triage_test.sh` | gate scripts on small inputs | python3 ≥ 3.11 |
| `scripts/check-fdev-verify-merge_test.sh` | verify-merge gate | cargo, fdev with `verify-merge` |
| `cargo test -p web-container-sign` | signer formats, contract acceptance | cargo |

## 7. Changing it safely

- Any edit under `scripts/`, `.github/`, `.githooks/`, `published-contract/`, to `Makefile.toml`, a `Cargo.toml`, `Cargo.lock`, a `.cargo/` file or `rust-toolchain.toml` runs every guard on the PR. A candidate rebuild is such an edit, so every delegate, protocol, wire-v1 or inbox-v2 change that alters bytes pays for all eight shards; only a UI-only change gets a scoped selection.
- Guards anchor on exact source text that must occur once. Reformatting, renaming or duplicating an anchored line breaks the guard ("mutation not applied"). Run `--anchors` after touching any file in the list in section 2.
- `Makefile.toml` is parsed textually by `release-freeze_test.sh:67-80` (first dependency of three tasks) and `check-release-scrub_test.sh:108-196` (exact lines; `FREENET_MAIL_SCRUB=1 ` must appear exactly twice; each ISO variable in exactly one file).
- `check-contract-wasm.yml` is parsed by `check-guard-shards.sh`; keep the matrix line, the `/8` step, the two `if:` lines and the `NIGHTLY_REPEAT` count.
- A new guard on a file under a new directory needs an `area_of` entry; a new `include_*!` or `fs::` read in a guarded crate must resolve in `select-guards_test.sh` or be added to `EXEMPT`.
- A new file under `contracts/inbox-v2/tests/` must be listed in `.cargo/mutants.toml` (`inbox-v2-gates.yml:91-98`).
- Rebuild order after an inbox-v2 change: `inbox_v2` and its hash first, then `mail_delegate` (it embeds that hash), then run delegate and UI tests.
- Freezing v2 will need: new names in `check-frozen-artifacts.sh:43-50` (unknown files fail L104-113), rows in `frozen.sha256`, new include paths in the UI and in `inbox_jobs.rs:84`, registry rows, removal of `.release-freeze`, and a workspace version bump.
- Toolchain bump: `rust-toolchain.toml` and ten `dtolnay/rust-toolchain@1.99.1` refs together; candidates will change bytes.
- `build-wasm.sh` builds the working tree, not HEAD: untracked non-ignored files are in the build.

## 8. Docs and comments that disagree with code

1. AGENTS "One-time setup": the hook "requires each committed contract WASM to be staged with its id file". No such check exists (`.githooks/pre-commit:38-67`).
2. AGENTS "The UI embeds `published-contract/wasm/v1/*`": it also embeds both candidates (`ui/src/mail_delegate/node.rs:19-22`, `ui/src/v2/mod.rs:27-29`).
3. `published-contract/candidate/README.md:15` "Nothing embeds it yet" (inbox_v2): the UI embeds it and the delegate embeds its hash.
4. Same README L18-19 and AGENTS name delegate, `protocol/`, `wire-v1/` as rebuild triggers; an inbox-v2 change is one too (`inbox_jobs.rs:84`).
5. Same README L8 says the release "moves" candidates into `wasm/v2/`; `check-release-delegate.sh:82-96` requires a build without `v1-transition`, so the delegate's frozen bytes cannot be the candidate's.
6. `e2e-real-node.yml:3-6` and AGENTS ("not every PR") against `pull_request:` with no filter (L18).
7. `RELEASING.md:224` and `smoke-test-production.sh:33-34,90-93` name `cargo make test-ui-live`; no such task (it is `test-e2e-real-node`). RELEASING also says "real RSA keygen".
8. `release-tag.sh:4-8` says `release.sh` opens a snapshot PR; it does not (`release.sh:353-354`).
9. `build-wasm.sh:12`, `delegates/mail-delegate/Cargo.toml:11,27` and `dependabot.yml:73` say the build is `--locked`; the invocation passes no such flag (`build-wasm.sh:99-102`). Whether fdev adds it cannot be seen from the tree.
10. `published-contract/README.md:93-95` "any machine reproduces the bytes" against `build-wasm.sh:16-17,59-60` and `candidate/README.md:29-32` (linux/amd64, CI canonical).
11. AGENTS and `check-container-byte-equal.sh:15-16` say the container build embeds the checkout path; the strings in `container.wasm` show only `/home/runner/.cargo/registry/…` (CARGO_HOME). A hidden dependence through the `../facade-types` path dependency is possible.
12. `RELEASING.md:147-170` puts the pointer flip before the frozen check and omits the delegate check and the default-on real-node E2E; code order is `release.sh:152`, `226`, `291`, `312`.
13. `test-contract/README.md:9-20` omits `mail-local-state/`; `identity/README.md` says the key has no bearing on production and gives a regeneration recipe, while `published-contract/README.md:45-48` says the shipped v1 params were derived from these keys.
14. `.release-freeze:5-8` says scrub, background jobs and inbox v2 "are still to land"; all three are in the tree.
15. `Makefile.toml:908` and AGENTS: "every out-of-workspace crate"; `check-excluded-crates.sh:12-23` omits `contracts/facade`, `contracts/facade-types` and `tools/inbox-v2-cost/{guest,host}`.
16. AGENTS describes a `prev_app_ids` ring; `sign-container-state` never passes `--prev` (`Makefile.toml:379-385`), so it is always empty.
17. `RELEASING.md:403` "release an older commit" against `release.sh:115-118` (HEAD must be `origin/main`).

## 9. Suspected bugs and weak spots

- **Fork CI cannot run as written (high).** Every heavy job is `runs-on: ubicloud-standard-8`. Without the Ubicloud app on the fork's account those jobs stay queued; only `changes`, `inbox-v2-gates.yml` and `inbox-v2-laws.yml` use GitHub-hosted runners. `check-drift` then cannot succeed.
- **Container byte-equality may fail in a fork with another repository name (medium-low).** If the checkout path feeds the bytes, as the repo's own comments say, `/home/runner/work/mail-freenet/mail-freenet` differs from upstream's `/home/runner/work/mail/mail`.
- **`build-wasm.sh` passes no `--locked` (medium).** A lockfile the manifests no longer satisfy would be re-resolved silently inside the throwaway stage.
- **macOS release from v0.3.0 (medium).** `release.sh:152` rebuilds the delegate through `build-wasm.sh` (flock, linux/amd64 canonical) while RELEASING still gives macOS setup.
- **Release check covers only `ui/` (medium).** `check-release-delegate.sh:72` does not see the candidate include in the delegate (`inbox_jobs.rs:84`), and nothing checks that a frozen `inbox_v2` exists or matches source.
- **`dbt` is defined and never used** (`check-guards-fail.sh:907`): no guard is backed by the unreadable-key run, yet its baseline runs in every full shard (L209). High confidence, low impact.
- `test-e2e-*` tasks build and sign the UI twice (task deps, then nested `cargo make publish-email-iso`). Waste only.
- `publish-container` depends on `build-container` (`Makefile.toml:438`) though it publishes the committed WASM.
- `release.sh:363-369` prints smoke URLs without `/v1/`.
- `e2e-real-node.yml:204-205` uploads the test nodes' `data/secrets/` as artifacts (test keys only).

## 10. Open work

- `.release-freeze`; no `wasm/v2/`; `v1-transition` still on in the candidate and in `ui/Cargo.toml:41,127`.
- `cargo make test-e2e-inbox-v2` runs in no workflow.
- `tools/web-container-sign/tests/facade_integration.rs:71-88` is a placeholder.
- `contracts/facade-loader/README.md:32-36`: Phase 2 loader (rollback to `prev_app_ids`) deferred.
- `check-guards-fail.sh:911-917`: one retired guard, documented.
- `check-guards-fail-inbox.sh:5-8`: real coverage, mutants and fdev runs are not repeated in the shards.
- No shellcheck or workflow lint in CI.

## 11. Questions the tree cannot answer

1. Which checks besides `check-drift` does upstream's ruleset require (`build-and-test`, e2e)?
2. Does `fdev build` pass `--locked` to cargo?
3. Does the container build depend on the checkout path, and do Ubicloud runners use `/home/runner/work/<repo>/<repo>`?
4. Is Actions enabled on the fork, and is Ubicloud (or a `runs-on` change) acceptable there?
5. Does a local `build-wasm.sh` run reproduce the committed candidates on this machine? (fdev, dx, cargo-make, xxd, b3sum and cbor2 are not installed here; cargo 1.99.1, flock, GNU tar, node 22, perl, openssl and python3 are.)

## Checklist before pushing

Common to all: `git config core.hooksPath .githooks` once; `cargo fmt --all -- --check` (rustup 1.99.1).

**(a) UI only (`ui/**`, no manifest change)**
1. `cargo clippy --workspace --all-targets -- -D warnings` (cargo).
2. `cargo test --workspace`, then `cargo test -p freenet-email-ui --lib --no-default-features --features example-data,no-sync` (cargo; no fdev).
3. `bash scripts/check-guards-fail.sh --anchors` (python3, git, GNU tar).
4. If you added a file read or include: `bash scripts/select-guards_test.sh`.
5. If you touched `scrub.rs`, `jobs.rs`, `ghostkeys.rs` or `lib.rs` marker lines: `scripts/check-release-scrub_test.sh`.
6. Optional, as CI: `cargo make build` (cargo-make, dx 0.7.10, wasm32 target); `cargo make test-ui-playwright` (plus Node 20, Playwright browsers, lsof, curl); selected guards with `printf '%s\n' <paths> | scripts/select-guards.sh pull_request > sel && scripts/check-guards-fail.sh --only sel --no-wasm`.

**(b) Mail delegate, `protocol/` or `wire-v1/`**: all of (a), plus
1. `scripts/check-excluded-crates.sh fmt`, `clippy`, `test` (cargo; network for ten lockfiles).
2. Rebuild the candidate (linux/amd64, rustup 1.99.1 with wasm32, fdev 0.3.313 from freenet-core v0.2.151, flock, GNU tar):
   `scripts/build-wasm.sh delegates/mail-delegate delegate freenet-main-delegate,v1-transition published-contract/candidate/mail_delegate | sed -n 's/^code hash: *//p' > published-contract/candidate/mail_delegate_code_hash`
3. `scripts/check-delegate-candidate.sh`; `scripts/check-delegate-imports.sh`; rerun delegate and UI tests (they read the candidate).
4. CI's sweep, in `delegates/mail-delegate`: `FAULT_SEEDS=2000 FAULT_OPS=80 FAULT_RATE=200 FAULT_BLIND_RATE=20 cargo test --release --locked --lib --features v1-transition tests::faults::invariants_hold_while_a_key_stays_unreadable -- --exact`.
5. `protocol/src/golden_v1.txt` and `wire-v1/src/golden_v1.txt`: append only. `cargo make check-golden-capture` for wire-v1.
6. Guards: all of them run in CI. Locally at least `GUARD_ONLY='^jobs: ' scripts/check-guards-fail.sh --no-wasm` or the relevant subset.
7. Optional: `cargo make test-e2e-background` (about 20 min; freenet and fdev v0.2.151, dx, Node, Playwright chromium, python3, perl, openssl, xxd).

**(c) Inbox v2 contract**: all of (b) with this order: rebuild `inbox_v2` first
   (`scripts/build-wasm.sh contracts/inbox-v2 contract contract published-contract/candidate/inbox_v2 | sed -n 's/^code hash: *//p' > published-contract/candidate/inbox_v2_code_hash`), then `mail_delegate`. Then:
1. `scripts/check-inbox-candidate.sh`; `scripts/check-delegate-imports.sh published-contract/candidate/inbox_v2`.
2. `cargo test --locked --manifest-path contracts/inbox-v2/Cargo.toml --features plausible-mutants --test metadata_disclosure --test plausible_mutants`; update `docs/metadata-fields.toml` for any new plaintext field.
3. `VECTORS_PYTHON=<venv python with cbor2 5.8.0> scripts/check-inbox-vectors.sh` (b3sum 1.8.5). Never regenerate `vectors.txt` to pass.
4. Slow gates: `COVERAGE_NIGHTLY=nightly-2025-08-08 FORK_SEEDS=3 scripts/check-branch-coverage.sh` (cargo-llvm-cov 0.6.21, llvm-tools-preview); `cargo mutants` in the crate, then `scripts/check-mutants-triage.sh --complete …/missed.txt …/timeout.txt` (cargo-mutants 27.1.0); `scripts/check-fdev-verify-merge.sh`; `scripts/check-inbox-cost.sh`.
5. Optional: `FORK_SEEDS=1000 cargo test --locked --test merge_laws`; `cargo make test-e2e-inbox-v2` (not in CI).

**(d) Scripts or workflows**
1. `bash scripts/select-guards_test.sh`; `bash scripts/check-guard-shards.sh`; `bash scripts/check-guards-fail.sh --anchors`.
2. Every `*_test.sh` for the script you touched (table in section 6).
3. `scripts/check-frozen-artifacts.sh` (fdev); `scripts/check-frozen-immutable.sh origin/main` (git).
4. If `build-wasm.sh` or `check-wasm-paths.sh` changed: `scripts/check-wasm-builds.sh` and both candidate checks.
5. Expect all guards in CI. Keep the shard matrix, the `dtolnay` refs and the parsed Makefile lines intact.

---

## Appendix: the reader's structured summary

This subsystem decides which bytes ship and proves they are the reviewed ones. Nothing was built or run for this map; it is from reading every owned file.

Three artifact classes exist. Frozen ones (webapp and container contracts, wasm/v1) are pinned by sha256 and never rebuilt. Candidates (mail_delegate, inbox_v2) must be rebuilt by the contributor with scripts/build-wasm.sh on every source change, and CI rebuilds and byte-compares them. Per-release state is signed by scripts/release.sh with an uncommitted key and is blocked today by .release-freeze.

About 417 guards (grep count) each apply one exact-text mutation to a staged copy and require a named test to fail. They run in eight CI shards; a pull request runs only the guards its diff can reach, unless it touches scripts, workflows, manifests or published-contract/, which runs all of them. A candidate rebuild is such a touch.

For the fork, the main finding is that CI cannot run as written. Every heavy job (build and test, drift checks, guard shards, both real-node suites) uses the third-party runner label ubicloud-standard-8. Only the selection job, the inbox v2 gates and the nightly merge laws use GitHub-hosted runners. No workflow needs a repository secret.

Several docs lag the code: the pre-commit hook lacks a check AGENTS.md describes, the UI and the delegate embed the candidates although READMEs say otherwise, and RELEASING.md names a cargo-make task that does not exist. The map ends with a per-change-type local checklist and the tools each step needs.

### Key facts

- Every heavy CI job runs on the third-party label ubicloud-standard-8: .github/workflows/build.yml:30,168; check-contract-wasm.yml:160,411; e2e-real-node.yml:43; e2e-background.yml:45. GitHub-hosted (ubuntu-latest) jobs are only changes/check-drift/report-nightly-failure (check-contract-wasm.yml:62,513,539), all of inbox-v2-gates.yml and inbox-v2-laws.yml.
- The only check named as required in the tree is check-drift (check-contract-wasm.yml:502-528); it succeeds only if changes, drift-checks and all 8 guard-shards report success, and is skipped only for docs-only PRs (L87-96).
- No workflow references a repository secret; only github.token is used, for gh run list (check-contract-wasm.yml:122) and gh issue (L547). The production key never enters CI.
- Triggers: build.yml and e2e-real-node.yml run on every PR (e2e-real-node.yml:18); e2e-background.yml and inbox-v2-gates.yml are path-filtered (e2e-background.yml:13-22, inbox-v2-gates.yml:15-24); inbox-v2-laws.yml is schedule/dispatch only.
- scripts/build-wasm.sh copies the working tree (tracked plus untracked-not-ignored, as on disk) to the fixed path /tmp/freenet-mail-wasm-build under flock (L61-78), remaps CARGO_HOME, RUSTUP_HOME, stage and target to /cargo, /rustup, /build, /target (L85-90), and runs fdev build (L99-102). It needs fdev, rustup, flock, GNU tar and Linux.
- check-delegate-candidate.sh:24-45 rebuilds with features freenet-main-delegate,v1-transition and cmp's against published-contract/candidate/mail_delegate plus its code hash; check-inbox-candidate.sh:23-56 does the same for contracts/inbox-v2 and also refuses clock or randomness imports (L34-39).
- The delegate source embeds the inbox v2 candidate's code hash (delegates/mail-delegate/src/inbox_jobs.rs:83-85), so an inbox-v2 change forces a rebuild of both candidates, inbox_v2 first. A delegate test also reads the committed delegate candidate (delegates/mail-delegate/src/tests/jobs.rs:173-177).
- The UI embeds both candidates today (ui/src/mail_delegate/node.rs:19-22, ui/src/v2/mod.rs:27-29) and the delegate candidate's code hash (ui/src/legacy_registry.rs:143).
- Guards: scripts/check-guards-fail.sh plus the two sourced files hold 416 mutation guards and one command guard by grep count. Each mutation is an exact string that must occur exactly once (L237-246); --anchors checks that without building (L286-296) and runs in drift-checks (check-contract-wasm.yml:322-323).
- scripts/select-guards.sh:26 HARNESS: a PR touching scripts/, published-contract/, .github/, .githooks/, .release-freeze, Cargo.lock, rust-toolchain.toml, Makefile.toml, any .cargo/ or any Cargo.toml runs every guard. A candidate rebuild is under published-contract/, so only UI-only PRs get a scoped selection.
- area_of (select-guards.sh:37-46) exits 1 for a guard whose mutated file is under an unmapped directory, and an area hit with an empty selection exits 1 (L73-76).
- scripts/check-guard-shards.sh parses check-contract-wasm.yml textually: matrix exactly 0..N-1 (L11-16), the --shard step (L17), check-drift needing guard-shards (L21), schedule-only nightly skip and exactly two NIGHTLY_REPEAT occurrences (L26-30).
- Makefile.toml is parsed textually by scripts/release-freeze_test.sh:67-80 and scripts/check-release-scrub_test.sh:108-196 (for example FREENET_MAIL_SCRUB=1 must appear exactly twice, L192), so task edits can break those tests.
- scripts/check-frozen-artifacts.sh hard-codes the five v1 artifact names (L43-50) and fails any other file in a wasm/v<N>/ directory (L104-113); it pins both production contract ids (L39-40).
- scripts/check-frozen-immutable.sh makes published-contract/wasm/, frozen.sha256 rows, legacy_*.toml rows and six fixture files append-only against the base commit (L69-144), including protocol/src/golden_v1.txt and wire-v1/src/golden_v1.txt.
- Release flow: release.sh runs the freeze gate first (L37), then preflight (L88-211), cargo make test/test-inbox/clippy (L216-218), test-e2e-real-node by default (L226), cargo make publish-production (L285), the published-contract/ unchanged check (L291), the container flip (L312-344) and release-tag.sh --sha (L354).
- check-release-delegate.sh passes below 0.3.0 (L66-69); from 0.3.0 it refuses any published-contract/candidate reference under ui/ (L72-77), a missing wasm/v2/mail_delegate (L78), the v1-transition marker (L82) and a byte mismatch with a fresh build using only freenet-main-delegate (L90-96).
- The pre-commit hook blocks changes to pinned files and anything under published-contract/wasm/ (.githooks/pre-commit:38-49) and any staged *.wasm other than the two top-level ones (L22, L52-66). Candidates are not covered. core.hooksPath is unset in this checkout.
- Tool pins used by CI: rustc 1.99.1 (rust-toolchain.toml:12), container rustc 1.95.0 (contracts/facade/rust-toolchain.toml), fdev and freenet from freenet-core v0.2.151 (build.yml:92, check-contract-wasm.yml:198), dioxus-cli 0.7.10 (build.yml:102), cbor2 5.8.0 and b3sum 1.8.5 (check-contract-wasm.yml:285-286), nightly-2025-08-08, cargo-llvm-cov 0.6.21, cargo-mutants 27.1.0 (inbox-v2-gates.yml:33-36).
- This sandbox lacks fdev, freenet, dx, cargo-make, xxd, b3sum, cbor2, cargo-llvm-cov and cargo-mutants; it has cargo 1.99.1, flock, GNU tar, node 22, perl, openssl, python3 and gh. No candidate rebuild, frozen-artifact check or UI release build can run here until those are installed.

### Docs or comments that disagree with the code

- **[low]** Claim (AGENTS.md, 'One-time setup'): The pre-commit hook requires each committed contract WASM to be staged with its id file. **Reality:** The hook only blocks changes to pinned or wasm/ files, changed manifest rows, and stray *.wasm files. There is no id-file check. Evidence: .githooks/pre-commit:38-67
- **[medium]** Claim (AGENTS.md, 'Frozen artifacts' first bullet): The UI embeds published-contract/wasm/v1/*. **Reality:** The UI also embeds both unfrozen candidates (mail_delegate and inbox_v2) and the delegate candidate's code hash. Evidence: ui/src/mail_delegate/node.rs:19-22; ui/src/v2/mod.rs:27-29; ui/src/legacy_registry.rs:143
- **[medium]** Claim (published-contract/candidate/README.md:15): inbox_v2: 'Nothing embeds it yet.' **Reality:** The UI embeds the inbox_v2 candidate and the mail delegate embeds its code hash. Evidence: ui/src/v2/mod.rs:27; delegates/mail-delegate/src/inbox_jobs.rs:83-85
- **[medium]** Claim (AGENTS.md (mail delegate section, last paragraph); published-contract/candidate/README.md:18-19): Rebuild the delegate candidate after changing the delegate, protocol/ or wire-v1/. **Reality:** An inbox-v2 change is also a trigger: the delegate includes candidate/inbox_v2_code_hash, so a new inbox candidate changes the delegate's bytes. Evidence: delegates/mail-delegate/src/inbox_jobs.rs:84
- **[medium]** Claim (published-contract/candidate/README.md:8): The release that ships the candidates moves them into wasm/v2/. **Reality:** The release guard requires wasm/v2/mail_delegate to be a build without v1-transition and without its marker, while the candidate is built with that feature. The frozen delegate cannot be the candidate's bytes. Evidence: scripts/check-release-delegate.sh:82-96; scripts/check-delegate-candidate.sh:24
- **[medium]** Claim (.github/workflows/e2e-real-node.yml:3-6 header comment; AGENTS.md 'Automated real-node harness', 'CI scope'): The real-node E2E is heavy and not run on every PR (tags, nightly, manual dispatch only). **Reality:** The workflow triggers on every pull_request with no path filter, and also runs the legacy-migration task. Evidence: .github/workflows/e2e-real-node.yml:12-18,181
- **[low]** Claim (RELEASING.md:224-230; scripts/smoke-test-production.sh:33-34,90-93): `cargo make test-ui-live` runs live-node.spec.ts against a fresh `freenet local` node with real RSA keygen. **Reality:** No task named test-ui-live exists. The task is test-e2e-real-node, on a two-node isolated `freenet network` harness. Evidence: Makefile.toml:1068-1131
- **[low]** Claim (scripts/release-tag.sh:4-8 header comment): release.sh publishes, then opens the published-contract/ snapshot PR; release-tag.sh tags the merged snapshot commit. **Reality:** release.sh opens no PR and commits nothing; it calls release-tag.sh --sha on the commit it built from. The snapshot-commit branch of release-tag.sh belongs to a superseded flow. Evidence: scripts/release.sh:288-296,353-354; scripts/release-tag.sh:97-131
- **[medium]** Claim (scripts/build-wasm.sh:12; delegates/mail-delegate/Cargo.toml:11,27; .github/dependabot.yml:72-75): build-wasm.sh builds with --locked. **Reality:** The script passes only `build --features ... [--package-type delegate]` to fdev. Whether fdev adds --locked cannot be seen from the tree. Evidence: scripts/build-wasm.sh:99-102
- **[low]** Claim (published-contract/README.md:93-95): build-wasm.sh remaps paths 'so any machine reproduces the bytes'. **Reality:** The script and the candidate README say to build on linux/amd64 with CI as the canonical builder, and the script needs flock and a fixed /tmp path. Evidence: scripts/build-wasm.sh:16-17,59-63; published-contract/candidate/README.md:29-32
- **[low]** Claim (AGENTS.md 'Status (#206 closed)'; scripts/check-container-byte-equal.sh:14-16): The container build embeds the checkout path, so only CI's checkout path reproduces the bytes. **Reality:** The strings in the committed container.wasm show only /home/runner/.cargo/registry/... (CARGO_HOME) and /rustc/... paths, no checkout path. A hidden dependence through the ../facade-types path dependency is possible but was not verified. Evidence: published-contract/container.wasm (strings output); contracts/facade/Cargo.toml:28
- **[low]** Claim (RELEASING.md:147-170): Release order: build/sign/publish, container pointer flip, then the frozen check; test gate is cargo make test, test-inbox, clippy. **Reality:** The published-contract/ unchanged check runs before the pointer flip. The runbook also omits the mail-delegate release check and the default-on test-e2e-real-node run. Evidence: scripts/release.sh:152,226-233,291-295,312-344
- **[medium]** Claim (test-contract/README.md:9-22; test-contract/identity/README.md (security section and regeneration recipe)): The committed test keys have no bearing on any production deployment; production must generate its own params. The directory holds web-container-keys.toml and identity/. **Reality:** published-contract/README.md says the shipped v1 identity-manager-params and mail-local-state-params were derived from these committed test keys. The directory also has mail-local-state/. AGENTS.md says not to rotate the identity key. Evidence: published-contract/README.md:45-48; test-contract/mail-local-state/README.md
- **[low]** Claim (.release-freeze:5-8): The scrub, the background jobs and inbox v2 'are still to land'. **Reality:** All three are in the tree; the freeze text lags. The freeze itself is still enforced. Evidence: ui/src/migration/scrub.rs:83; delegates/mail-delegate/src/inbox_jobs.rs:84; ui/src/v2/mod.rs:27
- **[low]** Claim (Makefile.toml:908; AGENTS.md 'Testing'): `cargo make test` runs the workspace, then every out-of-workspace crate. **Reality:** check-excluded-crates.sh lists ten crates and omits contracts/facade, contracts/facade-types and tools/inbox-v2-cost/{guest,host}. The facade is tested by a separate build.yml step. Evidence: scripts/check-excluded-crates.sh:12-23; .github/workflows/build.yml:138-144
- **[low]** Claim (AGENTS.md 'Container contract' and 'Pointer lifecycle' step 4): The container pointer keeps a ring of previous app ids (prev_app_ids=[...]) for rollback. **Reality:** The signer supports --prev, but the signing tasks never pass it, so every signed state has an empty prev_app_ids. Evidence: Makefile.toml:348-354,379-385; tools/web-container-sign/src/main.rs:101-102
- **[low]** Claim (RELEASING.md:403-404; .claude/skills/freenet-mail-release/SKILL.md (Recovery)): Roll back by releasing an older commit. **Reality:** release.sh refuses unless HEAD equals origin/main, so an older commit can only be released by reverting main first. Evidence: scripts/release.sh:115-118

### Suspected issues (leads from reading, not confirmed)

- **CI cannot run in the fork as written** (confidence: high). Build and test, drift-checks, all eight guard shards and both real-node suites use runs-on: ubicloud-standard-8. Without the Ubicloud app on the fork's account these jobs stay queued, so check-drift (which needs them) cannot succeed. Only the selection job, inbox-v2-gates.yml and inbox-v2-laws.yml use GitHub-hosted runners. Forks also have Actions and schedules off by default. The runner label is certain; the queueing behaviour is GitHub's, not something in the tree. Evidence: .github/workflows/build.yml:30; .github/workflows/check-contract-wasm.yml:160,411,510-528
- **Container byte-equality may fail in a fork with a different repository name** (confidence: low). The repo's own comments say cargo hashes the absolute path of out-of-workspace path dependencies into symbol names, and the container build has such a dependency (../facade-types) and is built without the fixed-stage trick. A fork checked out at /home/runner/work/mail-freenet/mail-freenet would then differ from upstream's /home/runner/work/mail/mail. Not verified by building; the visible strings in container.wasm depend only on CARGO_HOME. Evidence: scripts/build-wasm.sh:52-58; contracts/facade/Cargo.toml:28; scripts/check-container-byte-equal.sh:41-56
- **build-wasm.sh passes no --locked** (confidence: medium). Three places say the candidate build is locked, but the fdev invocation has no such flag. If a path dependency's requirement outruns the crate's lockfile, cargo could re-resolve silently inside the throwaway stage and the built bytes would follow an uncommitted resolution. The cmp would still fail closed, but the cause would be hidden. Evidence: scripts/build-wasm.sh:99-102
- **Release check for candidates covers only ui/ and only the delegate** (confidence: medium). check-release-delegate.sh greps ui/ for candidate references, but the delegate itself includes candidate/inbox_v2_code_hash. Nothing in the release path checks that a frozen inbox_v2 exists or equals the source's build. Evidence: scripts/check-release-delegate.sh:72-96; delegates/mail-delegate/src/inbox_jobs.rs:84
- **A v0.3.0 release probably cannot be cut from macOS** (confidence: medium). From 0.3.0, release.sh preflight rebuilds the delegate through build-wasm.sh, which needs flock and is only canonical on linux/amd64. RELEASING.md and check-release-delegate.sh's own comment still assume macOS release machines. Evidence: scripts/release.sh:152; scripts/build-wasm.sh:59-63; scripts/check-release-delegate.sh:43-44
- **check-frozen-artifacts.sh will reject a v2 generation until edited** (confidence: high). The script knows only the five v1 artifact names and fails any other file in a generation directory, so adding wasm/v2/mail_delegate or inbox_v2 fails the required check until the list is extended. Evidence: scripts/check-frozen-artifacts.sh:43-50,104-113
- **dbt helper is defined and never used** (confidence: high). No guard is backed by the unreadable-key fault run, yet its baseline runs in every full shard. Low impact: wasted CI time and a gap in what the guards show. Evidence: scripts/check-guards-fail.sh:907,209
- **Real-node tasks build and sign the UI twice** (confidence: medium). test-e2e-real-node, test-e2e-inbox-v2-run and test-e2e-background-run list build and derive-test-contract-id as dependencies and then call a nested cargo make publish-email-iso, which rebuilds, re-signs and re-derives. Waste only. Evidence: Makefile.toml:1070,1105,1145,1174,1209,1238
- **publish-container needlessly depends on build-container** (confidence: high). The one-time production container publish rebuilds the container from source with the 1.95.0 toolchain, then publishes the committed WASM. The rebuild is unused and adds a failure point. Evidence: Makefile.toml:438,443
- **release.sh prints smoke-test URLs without /v1/** (confidence: high). The next-steps text gives http://<gw>:<port>/contract/web/<id>/, while the gateway path used everywhere else is /v1/contract/web/<id>/. Evidence: scripts/release.sh:363-369; scripts/smoke-test-production.sh:55

### Open work

- Release freeze in force: .release-freeze:1-11, enforced by scripts/check-release-freeze.sh:27-31, scripts/release.sh:37 and Makefile.toml:438,839,863.
- No published-contract/wasm/v2/ exists; v0.3.0 must freeze the delegate (built without v1-transition) and inbox_v2: scripts/check-release-delegate.sh:28,78-96; Cargo.toml:38-40,47-50.
- v1-transition is still on in the candidate build and the UI's path dependency: scripts/check-delegate-candidate.sh:24; ui/Cargo.toml:41,127; delegates/mail-delegate/Cargo.toml (feature comment: removed when the v2 inbox lands).
- Workspace version is 0.2.7 (Cargo.toml:56) while release.sh requires it to equal the release version (scripts/release.sh:128-146).
- cargo make test-e2e-inbox-v2 (Makefile.toml:1133-1196) is run by no workflow.
- tools/web-container-sign/tests/facade_integration.rs:71-88: placeholder test that asserts nothing ('replace this with a real id-equality check').
- contracts/facade-loader/README.md:32-36: Phase 2 loader (subscribe to state, automatic rollback to prev_app_ids) deferred.
- scripts/check-guards-fail-inbox.sh:5-8: real branch-coverage, cargo-mutants and fdev runs are not repeated as guards; only their scripts are shown failing on small inputs.
- scripts/check-guards-fail.sh:911-917: one guard retired, with the reason recorded.
- No shellcheck or workflow lint runs in CI (no such step in .github/workflows/).
- tools/inbox-v2-cost/{guest,host} have no fmt, clippy or test coverage; they are only built by scripts/check-inbox-cost.sh:58-59.
- Documented limit: the container snapshot can only be regenerated on native linux/amd64 or taken from CI's artifact (scripts/build-container-snapshot-linux.sh:46-53; .github/workflows/check-contract-wasm.yml:365-372).

### Open questions

- Which checks besides check-drift does upstream's ruleset require (build-and-test, ui-playwright-tests, e2e-real-node)? Rulesets are not in the tree.
- Does `fdev build` pass --locked to cargo? The script does not, and three comments say the build is locked.
- Does the container build depend on the checkout path as well as CARGO_HOME, and do Ubicloud runners check out at /home/runner/work/<repo>/<repo>? This decides whether the byte-equality step can pass in a fork named mail-freenet.
- Is GitHub Actions enabled on one137th/mail-freenet, and is installing Ubicloud or changing runs-on (a fork-only divergence that runs every guard) acceptable there?
- Does a local build-wasm.sh run on a linux/amd64 machine outside CI reproduce the committed candidates byte for byte? Not testable here: fdev is not installed and building was out of scope.
- Is the fork's default token allowed issues: write and are Issues enabled? report-nightly-failure needs both, but only on a failed scheduled run.
- Will the candidate build keep v1-transition until the freeze, or is it removed first so the candidate and the frozen delegate can be the same bytes?

### Coverage

Read in full: `Makefile.toml`, `Cargo.toml`, `rust-toolchain.toml`, `.gitignore`, `.release-freeze`, `.githooks/pre-commit`, `.claude/settings.json`, `RELEASING.md`, `.github/dependabot.yml`, `.github/workflows/build.yml`, `.github/workflows/check-contract-wasm.yml`, `.github/workflows/e2e-background.yml`, `.github/workflows/e2e-real-node.yml`, `.github/workflows/inbox-v2-gates.yml`, `.github/workflows/inbox-v2-laws.yml`, `scripts/build-container-snapshot-linux.sh`, `scripts/build-loader.sh`, `scripts/build-wasm.sh`, `scripts/check-branch-coverage.py`, `scripts/check-branch-coverage.sh`, `scripts/check-branch-coverage_test.sh`, `scripts/check-container-byte-equal.sh`, `scripts/check-delegate-candidate.sh`, `scripts/check-delegate-imports.sh`, `scripts/check-delegate-imports_test.sh`, `scripts/check-excluded-crates.sh`, `scripts/check-fdev-verify-merge.sh`, `scripts/check-fdev-verify-merge_test.sh`, `scripts/check-frozen-artifacts.sh`, `scripts/check-frozen-immutable.sh`, `scripts/check-guard-shards.sh`, `scripts/check-guards-fail.sh`, `scripts/check-guards-fail-jobs.sh`, `scripts/check-guards-fail-inbox.sh`, `scripts/check-inbox-candidate.sh`, `scripts/check-inbox-cost.sh`, `scripts/check-inbox-vectors.sh`, `scripts/check-mutants-triage.sh`, `scripts/check-mutants-triage_test.sh`, `scripts/check-release-delegate.sh`, `scripts/check-release-delegate_test.sh`, `scripts/check-release-freeze.sh`, `scripts/check-release-scrub.sh`, `scripts/check-release-scrub_test.sh`, `scripts/check-wasm-builds.sh`, `scripts/check-wasm-paths.sh`, `scripts/container-toolchain.sh`, `scripts/delegate-imports-allowlist.txt`, `scripts/generate-production-key.sh`, `scripts/release-freeze_test.sh`, `scripts/release-tag.sh`, `scripts/release.sh`, `scripts/run-isolated-nodes.sh`, `scripts/select-guards.sh`, `scripts/select-guards_test.sh`, `scripts/smoke-test-production.sh`, `published-contract/README.md`, `published-contract/candidate/README.md`, `published-contract/frozen.sha256`, `published-contract/contract-id.txt`, `published-contract/container-id.txt`, `published-contract/candidate/mail_delegate_code_hash`, `published-contract/candidate/inbox_v2_code_hash`, `test-contract/README.md`, `test-contract/identity/README.md`, `test-contract/mail-local-state/README.md`, `contracts/facade-loader/README.md`, `contracts/facade-loader/src/index.html.tmpl`, `tools/web-container-sign/Cargo.toml`, `tools/web-container-sign/src/main.rs`, `tools/web-container-sign/tests/facade_integration.rs`, `ui/build.rs (outside owned paths, for the build interface)`, `contracts/facade/Cargo.toml (outside owned paths)`, `contracts/facade/rust-toolchain.toml (outside owned paths)`, `contracts/.cargo/config.toml and modules/.cargo/config.toml (outside owned paths)`, `.claude/skills/freenet-mail-release/SKILL.md (outside owned paths)`

Skimmed: `README.md (root; first 120 lines, outside owned paths)`, `ui/Cargo.toml (dependency and feature sections only)`, `delegates/mail-delegate/Cargo.toml (first 80 lines)`, `delegates/mail-delegate/src/inbox_jobs.rs, handlers.rs, tests/jobs.rs (only the lines that include published-contract files)`, `ui/src/mail_delegate/node.rs and ui/src/v2/mod.rs (only the include lines)`, `contracts/inbox-v2/.cargo/mutants.toml (first 40 lines) and contracts/inbox-v2/Cargo.toml (section headers)`, `tools/inbox-v2-cost/{guest,host}/Cargo.toml (workspace stanza and path deps only)`, `published-contract/*.wasm, wasm/v1/* and candidate binaries (strings and header bytes only, for embedded paths)`, `test-contract/web-container-keys.toml (layout only; secret value not read into the map)`

Skipped: `Cargo.lock and every crate-local Cargo.lock: lockfiles, excluded by the reading rule`, `published-contract/webapp.parameters and container.parameters: 32-byte binaries; xxd is not installed here, so only their sha256 rows were compared`, `published-contract/wasm/v1/*_code_hash and *-params: values are pinned in scripts/check-frozen-artifacts.sh; not opened individually`, `test-contract/identity/*.pem and test-contract/mail-local-state/*.pem: key material; only the layout was listed`, `.claude/skills/freenet-mail-qa/SKILL.md: outside owned paths (only settings.json is owned)`, `tools/v1-node-seed/ and tools/inbox-v2-cost/ sources: outside owned paths`, `ui/tests/*.spec.ts and ui/tests/playwright.config.ts: outside owned paths; only the file list and package.json were viewed`
