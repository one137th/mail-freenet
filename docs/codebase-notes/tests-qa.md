> Notes v1, 2026-11-19. Describes commit `2358b59`. Written by an AI reader from the source; nothing was built or run. See [README.md](README.md) for how far to trust this.

# Map: end-to-end tests and the QA inventory (key: tests-qa)

Paths are relative to the repo root. `L` = line.

## 1. Purpose and where this sits

`ui/tests/` is one Playwright project (15 spec files, 142 `test(...)` declarations) that drives the Dioxus UI in four environments:

| Environment | Build | Node | Specs |
|---|---|---|---|
| offline | `dx serve --features example-data,no-sync --no-default-features` (dev profile; real mail delegate linked into the page via `local-delegate`, test Ghost Key vault via `test-ghost-master`: `ui/Cargo.toml` `[features]`) | none | email-app, threads, threads-storybook, compose-autocomplete, inbox-v2, inbox-v2-ghost, migration-ui, mail-delegate-identity, export-permission |
| iso (real nodes) | `cargo make build` (`use-node` release, plus one extra feature per task) published to a 2-node isolated network | gateway + peer from `scripts/run-isolated-nodes.sh` | live-node, legacy-migration, background-proof, inbox-v2-live, repro-106-107 (manual only) |
| deployed | whatever is published | a gateway URL | production-liveness |

`docs/qa/manual-test-inventory.md` is the declared source of truth for auto vs manual coverage (AGENTS.md:1001); `.claude/skills/freenet-mail-qa/SKILL.md` tells agents how to use it; `.claude/skills/freenet-mail-release/SKILL.md` is an agent runbook around `scripts/release.sh`. None of these four ships to users. Nothing here is frozen. The inventory and both skills are materially stale (section 8).

## 2. Walkthrough

### Config and shared modules

- `ui/tests/playwright.config.ts`: `testMatch "*.spec.ts"` (L5); `testIgnore` drops `legacy-migration`, `background-proof`, `inbox-v2-live` unless `FREENET_LEGACY_SEEDED` / `FREENET_BACKGROUND_PROOF` / `FREENET_INBOX_V2_LIVE` is `"1"` (L11-17); test timeout 120 s (L18); `retries: 2` (L19); `baseURL = FREENET_EMAIL_BASE_URL || http://127.0.0.1:8092` (L24); navigation 30 s, action 10 s (L25-26); `trace: "on-first-retry"` (L29); two projects only, `chromium` (Desktop Chrome) and `mobile-chrome` (Pixel 5) (L31-40). No `webServer`: the server must already run (L44-45). No reporter, workers or `fullyParallel` set.
- `ui/tests/package.json`: `@playwright/test ^1.50.0`; lockfile resolves 1.59.1 (`package-lock.json:13`), which the CI browser cache key also names (`.github/workflows/build.yml:232`). No `tsconfig.json`, so specs are never type-checked.
- `ui/tests/testids.ts`: `TID` = `JSON.parse(ui/branding/testids.json)` (L169-171). The `TestIds` interface (L12-167) is hand-written and has drifted: 15 JSON keys are missing from it (`fmSettingsBtn`, `fmSettingsShell`, `fmSettingsBack`, `fmSettingsNavItem`, used by five specs, plus `fmMsgTime`, `fmQuarantineCard`, `fmDetailTime`, `fmDetailFingerprint`, `fmDetailVerif`, `fmComposeSendingAs`, `fmSidebarFingerprint`, `fmSidebarContacts`, `fmSidebarVersion`, `fmComposeRecipientHint`, `fmThreadingEnabledToggle`) and one member has no JSON key (`fmArchiveUnarchive`, L55). It works only because nothing type-checks.
- `ui/tests/app-name.ts`: `APP_NAME` from `ui/branding/app.json` (`"Mail"`).

### Offline specs

All navigate to `/`, wait for `.brand-name`, and open a seeded identity (`address1`, `address2`). Seed: thread "Lunch tomorrow?" (ids 0,1,2) plus standalone ids 3,4 on address1; legacy "Picnic this weekend?" thread and one Sent row on address2; verified contact `alice-example` (`threads.spec.ts:9-21`, `ui/src/app.rs:1010`).

- `email-app.spec.ts` (2956 lines, 74 tests). Helpers L11-81 (`openNav` opens the mobile drawer before every sidebar click). Groups: identity list L85; inbox shell, threaded detail, empty folders L103-188; Quarantine toggle L190-256; compose and logout L258; Send dismisses sheet L307; "Sent" toast auto-dismiss (#290) L342; identity list survives reload (#36) L383; two-identity messaging L411; cross-identity threaded reply L515; app inside a sandboxed iframe L656 (hard-codes the 8092 fallback, L666); no horizontal overflow at 1280 px L694; contact import, compose badge, Remove L725-830; Drafts (5 tests) L836-987; create-alias reveal and duplicate alias (#52, #117) L997-1049; share modal L1051; verify on import and Verify button (#87) L1084-1185; restore form (6 tests) L1312-1497 using `FIXTURE_TOKEN` (L1190) and `ui/src/app/testdata/identity-backup-v2.json` (L1192); recovery-key backup panel (14 tests) L1499-2041 with clipboard shims `breakLocalClipboard`/`slowClipboard`/`manualClipboard` (L1236-1300); gateway shell relays (#349) with a stand-in shell page and a cross-origin negative L2047-2161; Sent folder, Reply/Forward/Resend prefill L2166-2322; rename L2327; delete identity L2381-2430; Archive (6 tests) L2435-2628; timestamps under a fixed clock (`TIMESTAMPS_TEST_NOW`, default `2026-06-15T12:00:00Z`) L2630-2686; trust pill, "Sending as" L2688; sidebar fingerprint L2729; delivery state L2769; bs58 card (#249) L2807; contact modals inside the inbox (#158) L2868. Skips: clipboard paste test off Chromium (L1409); two Settings identity-switcher tests on mobile (L1949, L1997).
- `threads.spec.ts` (10 tests): grouping and count badge L121; open detail L163; Nested default expansion L188; quote chip L223; touch action cluster CSS per `(hover: none)` (#317) L269; Compact view via `fm-thread-view-select` L307; reply stays in thread L368; legacy two-party fold (#287) L448; Sent fold as `.is-me` (#310) L508; Sent delete L551.
- `threads-storybook.spec.ts` (4 tests): `/?story=sender-sent-in-thread | deep-nest | nested-vs-compact&view=nested|compact`; asserts `.is-me`, depth cap 4 (`ui/src/app.rs:1593`), row counts; screenshots to the output dir.
- `compose-autocomplete.spec.ts` (9 tests): To-field dropdown on `alice-example` (click, ArrowDown+Enter, Escape, verified mark) and a Pixel 5 tap (L207-266).
- `inbox-v2.spec.ts` (9 tests, L180-515): address typo, v0.2 address, non-ASCII label, own identity refused, lookalike label warning (L181); `ghost-5` badge and refusal, Open queue, accept with petname, reply by pass, delete, pass held (L217); invite (L308); `/?v2-foreign-fixture` "active on another node" (L334) and its negative (L508); archive (L350) and delete (L394) of a drained message via `data-v2-stored`; lookalike stranger flagged (L423); "Add to address book" uses the v2 flow (L482). Each asserts an empty console.
- `inbox-v2-ghost.spec.ts` (5 tests): vault in localStorage `fmail-ghostkeys-mock` (L21; fixture indices 0-3 $1, 4-7 $5, 8-9 $20, L7-8); consent once (L134); picker and below-tier refusal (L172); declined prompt, no key at send (L195); "Get a Ghost Key" link `https://freenet.org/ghostkey/create/` with `return_to=GeQLXXR8sfGe31BNRJoVxkvGUTnBkufJ4gTaoX5NWnU5` (the production webapp id) and `return_path=#/compose?draft=...` (L236-241); `/?v2-queue-fixture` full queue of 16, drop, accept (L262).
- `migration-ui.spec.ts` (7 tests): `/?migration-fixture=not-installed|silent|data` (`ui/src/app/migration_fixture.rs:21-27`); waits for `fm-migration-report`; notice text, backup from notice, orphan move/discard with confirmation, confirm-identity gate before share (L98) and before send (L119).
- `mail-delegate-identity.spec.ts` (2 tests): key shown at creation equals a later export and leaves the DOM on close (L39); delete then restore by key gives the same fingerprint (L73).
- `export-permission.spec.ts` (1 test): Settings → Account → revoke remembered export grant → toast.

### Real-node specs (all `retries: 0`, Chromium only)

- `live-node.spec.ts` (3011 lines, 16 tests, one `test.fixme`). Gate: `FREENET_EMAIL_BASE_URL` must contain `/v1/contract/web/` (L193-197); project must be `chromium` (L206-209); `beforeEach` requires `GET {gw}/permission/pending` to answer (L211-212). Evidence wrapper (L97-190) patches `browser.newContext`/`ctx.close` to keep console, screenshot and DOM, written only on failure, because trace-on-retry never fires with 0 retries (L216-218). Tests: T1 create → reload → delegate compiled → no `secret not found` → backup panel copy and download through the shell proxy (L223-436); alice→bob cross-node (#81) L449; node holds its own inbox (#288) L662; multi-round, click-to-read survives reload, archive L739; reply after add-from-message under another nickname (#289) L1105; contact import + reload + UpdateNotification (#204) L1315; verified-skip toggle (#157) L1521; tier retune (#221) L1650; Sent survives reload (#286) L1868; delete survives reload, with Bob's WebSocket held by `routeWebSocket` to prove "Deleting…" vs "Deleted" (#286) L2029-2233; fixme delete+regen (#5/#291) L2258; two tabs, compare-and-swap settings merge L2331; rename then draft+settings persist L2409; draft edited inside the 500 ms debounce is saved L2497; identity left at the reveal step self-heals its contracts L2560; hub action buttons do not move during load L2671. Helpers: `composeAndSend` L2751, `createIdentity` (one reload fallback) L2779, pump L2831, overlay counter L2873, log helpers L2904-3011.
- `legacy-migration.spec.ts` (1 test, 480 s): seeded v0.2.7 identity `alice` and archive arrive; unconfirmed identity must confirm before send, send reaches `Delivered`; backup from the notice; reload until the report contains `"scrubbed":{"1000206":true,"206":true}` (L178-183); data still present. Throws (not skips) under `CI` when it cannot run (L33-38).
- `background-proof.spec.ts` (1 test, 60 min): Background permission asked exactly once per node (L271-290, L387-390); waits `FIRST_FIRE_WINDOW_S = 90`; cost run imports 3; notification imports a 4th with tabs closed inside the no-rescan window `NO_RESCAN_UNTIL_S = 900`; a 5th is imported by the `rescan` wake-up after `WAKEUP_WAIT_S = 1070` (L49-51); asserts on `data-jobs-run` (`imported`, `kept`, `verified`, `scanned == 4`, `titles`) L403-423 and no `Lifecycle run failed` in either log (L381-386). Writes `PROOF` lines and `BACKGROUND_PROOF_OUT`. Throws under `CI` when it cannot run (L250-253).
- `inbox-v2-live.spec.ts` (1 test, 60 min): Alice/Carol/Dave on the gateway, Bob on the peer; queue → accept → passes both ways → invite → revoke (`data-state` live → revoked) → RequireGhost $5 refused then admitted with the test vault → tab-closed drain. Drain must be by notification within `NOTIFY_BOUND_MS = 120_000` and the peer log must name `woken_by=notification` (L409-412); otherwise it waits to `TAB_CLOSED_WAIT_MS = 1_100_000` after the send for `rescan` and then fails (L45-47, L387-390). No `CI` guard: only `test.skip(!CONTRACT_ID)` (L244).
- `repro-106-107.spec.ts` (1 test): diagnostic two-node repro with soft assertions; run by hand only (L1-15); skipped unless the base URL carries a contract id (L154).
- `production-liveness.spec.ts` (1 test): inside `iframe#app`, brand shows `APP_NAME`, "Create new identity" visible, `h1.display` computes `font-style: italic`, no console errors except one allow-listed wasm-bindgen line (L104-106). Skips unless the base URL contains `/v1/contract/web/` (L35-41).

### The permission pump

Canonical form `startPermissionPump(origin)` (`live-node.spec.ts:2831-2865`): every 200 ms `GET {origin}/permission/pending` with an `Origin: {origin}` header; the JSON array's items carry `nonce` (also `message`, `labels`, L2814-2818); for each unseen nonce `POST {origin}/permission/{nonce}/respond`, body `{"index":0}` (first button = Allow), JSON content type and `Origin`; errors are swallowed. One pump per node, stopped in `finally`. It answers every prompt: AFT token, delegate consent (export, delete), Background. No spec exercises a denial on a real node.
Variants: `legacy-migration.spec.ts:50-85` is strict (anything but a refused connection or 5xx, or more than 5 busy ticks in a row, sets `pumpError`, asserted null at L208); `background-proof.spec.ts:78-102` records each prompt body so the test can count Background prompts; `inbox-v2-live.spec.ts:54-75`; `repro-106-107.spec.ts:31-58` sends no `Origin` header.
Overlay race (#367): the shell's `#__freenet_perm_overlay` can swallow a click aimed into the app frame. `countHostPrompts`/`hostPromptsClosed` (L2873-2897) count overlay hides in the top frame; the Sent test waits for one before clicking a folder (L1956-1961). `background-proof.spec.ts:208-219` waits on `data-pending-saves` instead.

### Log assertions

`grepLogDir` reads only the last 256 KiB of each `freenet.*` file (`live-node.spec.ts:2983-3011`); `grepLog` = gateway dir, `grepPeerLog` = peer dir; `logHasLineWith` reads whole files and needs all needles on one line (L2959-2981). Nodes log at `--log-level info` (`scripts/run-isolated-nodes.sh:202, 255`).
- Hard: `Module cache miss` plus the hash in `published-contract/candidate/mail_delegate_code_hash` (L289-314); no `secret not found` (L323); no receiver errors `missing field \`tier\`|delta_apply_failed|slot.*collision|failed to deserialize.*Inbox|InboxUpdateError` on the peer (L621-627); no `task .* panicked|missing related contract` on either node (L628-635); no `Rejecting SUBSCRIBE: contract WASM not cached locally` on the peer (L686-699).
- Diagnostic only: `phase="update_complete"|phase="relay_update|update_broadcast` (L555-563).
- Quarantine: when cross-node delivery times out, `coreRelayBugDetected()` (regex L2949-2950) and no "genuine regression" regex in the peer log turn the failure into `test.skip` (L596-606, L893-903, L1203-1211, L2126-2134).
- UI-side markers are read from the browser console, not node logs: `primed contact inbox … (#191)` L1349/L1400, `UpdateVerifiedBypass dispatched … allow=true (#157)` L1528, `contact tier cache: recorded … tier=Min1` L1684, `send: contact-tier-cache hit` L1692, `UpdateInboxPolicy dispatched … (#85)` L1698, self-heal lines L2599-2664, `none unanswered` L2223.
- `inbox-v2-live.spec.ts:216-241` parses `mail delegate run: woken_by=… imported=N` from `freenet.YYYY-MM-DD*` peer files only.

### Docs and skills

- `docs/qa/manual-test-inventory.md` (307 lines): status vocabulary L9-16; suites L21-66; the "persistence is iso-only" rule L68-89; matrix L91-284 (Identity, Contacts, Inbox, Threading, Compose, Drafts, Sent, Archive, Settings, Layout, Network); gap list L286-298; usage L300-307.
- `freenet-mail-qa/SKILL.md` (95 lines): when to invoke, per-status actions L19-29, suite commands L46-53, "WIP settings gating" L58-83, anti-patterns L85-95.
- `freenet-mail-release/SKILL.md` (214 lines): can/cannot L20-46, preflight commands L50-78, the three `release.sh` prompts L106-110 (match `scripts/release.sh:251, 329, 353`), post-release probes L127-145, recovery L151-184.

## 3. Interfaces

- Env vars: `FREENET_EMAIL_BASE_URL`; `FREENET_ISO_GW_PORT` / `FREENET_ISO_PEER_PORT` (spec defaults 7510/7511; the make tasks pick free ports, `Makefile.toml:1081-1087`); `FREENET_ISO_GW_LOG_DIR` / `FREENET_ISO_PEER_LOG_DIR` (default `~/freenet-mail-iso/{gw,peer}/logs`, `live-node.spec.ts:39-44`); the three `testIgnore` gates; `CI`; `BACKGROUND_PROOF_OUT`; `BACKGROUND_WAKEUP_WAIT_S`; `TIMESTAMPS_TEST_NOW`; task-level `E2E_PLAYWRIGHT_ARGS`, `BACKGROUND_PLAYWRIGHT_ARGS`, `PLAYWRIGHT_ARGS`, `FREENET_EMAIL_DX_PORT`, `FREENET_E2E_KEEP`, `FREENET_E2E_ROOT`.
- Gateway HTTP: `/permission/pending`, `/permission/{nonce}/respond`, `/v1/contract/web/{id}/`; shell DOM `iframe#app`, `#__freenet_perm_overlay`; shell messages `{__freenet_shell__: true, type: "download"|"clipboard", …}` (`live-node.spec.ts:351-372`, `email-app.spec.ts:2087-2106`); WebSocket path `/v1/contract/command` (L2055).
- DOM hooks the UI exposes for tests: `data-pending-saves` on `<html>` (`ui/src/pending_saves.rs:62`); `data-jobs-run` (`ui/src/jobs.rs:173`, `e2e-probe` only); `data-autosave` (`ui/src/app.rs:6599`); `data-v2-stored` (`app.rs:3891`); `data-seq`, `data-delivered` (`ui/src/app/v2_view.rs:905-906`); `data-delivery-state` (`app.rs:3962`); `data-share-text` (`ui/src/app/login.rs:2358`); `data-key-exported`, `data-copies-in-flight` (`login.rs:1757-1758`); folder buttons `fm-folder-{label}` built by `format!` (`app.rs:3651`), so not in the JSON.
- Test-only URL hooks (offline builds): `?story=`, `?migration-fixture=`, `?v2-foreign-fixture`, `?v2-queue-fixture` (`ui/src/v2/owner.rs:172-180`); localStorage `fmail-ghostkeys-mock` (`ui/src/ghostkeys.rs:826`).
- Testids: `ui/branding/testids.json` ↔ `ui/src/testid.rs`, pinned by `testid::tests::json_matches_rust_consts` (`testid.rs:246`). Specs also hard-code ids outside the JSON (`contact-row`, `contact-remove`, `contact-verify-badge`, `compose-recipient-fingerprint`, `fm-aft-tier`, `fm-settings-ident-switcher`).
- Other inputs: `published-contract/candidate/mail_delegate_code_hash`; `tools/v1-node-seed` (seed and `--expect-cleared`, `Makefile.toml:1328, 1336`); `scripts/check-release-scrub.sh`.

## 4. Invariants and where they are enforced

1. A test build must never reach a real node or production. `publish-email-iso` runs `run-isolated-nodes.sh check-ours` first (`Makefile.toml:812`; script L172-178). Each real-node task asserts the release check refuses its own build: scrub off (L1107-1114), test Ghost Key master (L1177-1184), probe (L1241-1248). `scripts/check-release-scrub_test.sh:128-171` greps these exact task lines and requires `FREENET_E2E_PROBE_ISO=1` / `FREENET_TEST_GHOST_ISO=1` to appear in exactly one file among `Makefile.toml`, `.github/`, `scripts/release.sh` (L143, L154-156). That test runs in `build.yml:165`.
2. Env-gated specs must not pass by skipping: `testIgnore` plus an explicit spec path makes Playwright error on "no tests" (`playwright.config.ts:6-10`), and two of the three throw under `CI` (above). `inbox-v2-live` does not.
3. Real-node specs never retry and use unique aliases (`live-node.spec.ts:57-96`). Identities accumulate: the delegate caps an origin at `MAX_IDENTITIES = 16` (`delegates/mail-delegate/src/store.rs:36`), and one full live-node run creates 14 on the gateway and 9 on the peer.
4. Offline cannot prove persistence (inventory L68-89): a reload reseeds. Any "survives reload" row is `auto` only with a real `live-node` test.
5. No fixed sleeps before a reload: wait for `data-pending-saves="0"` or `data-autosave="saved"` (`live-node.spec.ts:1980-1983, 2451, 2529-2532`).
6. Offline specs must pass on both viewports: `openNav` before sidebar clicks, `dispatchEvent("click")` and `state: "attached"` for the detail column, which collapses to zero width on Pixel 5 (`threads.spec.ts:58-68`).
7. Testid JSON and Rust consts agree (Rust unit test). Nothing pins the TypeScript interface.

## 5. State

Live and running in CI: the offline suite and live-node + legacy-migration (every PR), background-proof (path-filtered PRs and nightly). Present but in no workflow: `inbox-v2-live` (`cargo make test-e2e-inbox-v2` appears only in `Makefile.toml` and the scrub test). Unmaintained: `repro-106-107`. Releases are frozen: `.release-freeze` exists and `scripts/release.sh:37` refuses; the workspace version is 0.2.7 (`Cargo.toml:56`) and only `published-contract/wasm/v1` exists, so `check-release-delegate.sh` would also refuse a v0.3.0 release today.

## 6. Running the tests

| Suite | Command | Needs | CI |
|---|---|---|---|
| offline | `cargo make test-ui-playwright` (`Makefile.toml:971-1036`: release build, then `dx serve` on the first free port in 8082-8182 or `FREENET_EMAIL_DX_PORT`, readiness = `wasm/app.js` in the HTML, then `npx playwright test ${PLAYWRIGHT_ARGS}`) | `dx` 0.7.10, node, `cargo make test-ui-playwright-setup` once, `lsof`, `curl` | `build.yml` job `ui-playwright-tests` (L167-295): port 8082, readiness `id="main"`, Chromium only, artifacts on failure; every PR and push to main |
| live-node | `cargo make test-e2e-real-node` (L1068-1131): wipe, up, `publish-email-iso`, then `npx playwright test live-node.spec.ts --project=chromium` | `freenet` and `fdev` (CI pins v0.2.151), `dx`, GNU tar, python3, perl, openssl, xxd | `e2e-real-node.yml`: tags `v*`, every PR, nightly 04:30 UTC, dispatch (L8-22); 45 min timeout |
| legacy migration | `cargo make test-e2e-legacy-migration` (L1289-1337) | as above; builds with `FREENET_MAIL_SCRUB=1 FREENET_MAIL_SCRUB_ISO=1` | second step of the same workflow (L180-183) |
| background | `cargo make test-e2e-background` (L1198-1263), about 25 min, `e2e-probe` build | as above | `e2e-background.yml`: PRs touching the listed paths (L12-22), nightly 05:30; not required |
| inbox v2 | `cargo make test-e2e-inbox-v2` (L1133-1196), `test-ghost-master` build, up to about 25 min | as above | none |
| liveness | `scripts/smoke-test-production.sh [url]` → `npx playwright test production-liveness.spec.ts` (L85) | a reachable gateway, `ui/tests/node_modules` | none |
| by hand | `cd ui && dx serve --port 8092 --features example-data,no-sync --no-default-features`, then `cd ui/tests && npx playwright test` | | |

`cargo make test` runs no Playwright (`Makefile.toml:907-919`). `scripts/release.sh:226-233` runs `test-e2e-real-node` before tagging unless `FREENET_RELEASE_E2E=0`. Every workflow that runs Playwright uses `runs-on: ubicloud-standard-8`.

In the CI offline job all spec files load; live-node, production-liveness and repro self-skip because the base URL has no contract path.

## 7. Changing it safely

- Adding or renaming a testid: edit `ui/branding/testids.json` and `ui/src/testid.rs` together (the Rust test fails otherwise), then the `TestIds` interface by hand.
- Editing any `test-e2e-*` task or `publish-email-iso`: keep the lines `check-release-scrub_test.sh` greps; a `Makefile.toml` change also selects every guard shard (`scripts/select-guards.sh:26`).
- Putting inbox v2 in CI: call `cargo make test-e2e-inbox-v2` from the workflow. Writing `FREENET_TEST_GHOST_ISO=1` into `.github/` breaks the one-file rule.
- Adding gateway-side identities to `live-node.spec.ts`: two slots are left before the 16-identity cap; enabling the fixme test takes one.
- A change under `ui/tests/` selects the `ui` guards in `check-drift` (`select-guards.sh:32`, prefix `ui/`); a change to `docs/qa/` selects none (`select-guards_test.sh:43`). `ui/tests/background-proof.spec.ts` is itself in the background workflow's path filter.
- A change to user-visible strings breaks specs that match text: toasts ("Sent", "Deleted", "Deleting"), `NOT_BACKED_UP` (`email-app.spec.ts:1232`), restore errors, v2 outcome sentences, migration notice text, placeholders `alias or address`, `subject`, `e.g. Alice (work)`. A change to UI log lines breaks the console-marker tests listed in section 2.
- A change to the seed in `load_example_messages` breaks hard-coded ids and counts (`#email-inbox-accessor-0/3/4`, quarantine badge "2", three groups).
- Bumping Playwright: update the cache key string in three workflows and the hand-copied apt package list (`build.yml:245-257`).
- Offline specs run twice (two projects) with two retries, so a slow or flaky addition costs up to six executions.

## 8. Disagreements between docs/comments and code

See the structured return for the full list with evidence. The ones that change what a planner would do:
- "5 browser profiles" (AGENTS.md:1010-1012, release skill L130, `smoke-test-production.sh:20`) vs two Chromium projects; the config comment suggesting `--project=firefox --project=webkit` (L41-42) names projects that do not exist.
- AGENTS.md:1030 says serve on 8082; the config default is 8092, so the documented manual recipe gets connection refused unless `FREENET_EMAIL_BASE_URL` is set.
- `FREENET_LIVE_E2E_SEND=1` (AGENTS.md:1068-1070, inventory L28-29, QA skill L51) is read by nothing; the cross-node tests always run (`live-node.spec.ts:445-448, 740`).
- "Tags / nightly, not every PR" (AGENTS.md:1093-1094, QA skill L55-56, the workflow's own header L5-6) vs `on: pull_request` with no path filter (`e2e-real-node.yml:18`).
- Log markers `UPDATE_PROPAGATION` and `allocate_token` (AGENTS.md:1086-1088) are no longer asserted (`live-node.spec.ts:547-563`).
- The `wip-settings` cargo feature (inventory L232-236, QA skill L58-83) does not exist in `ui/Cargo.toml`; the skill's `cargo check --features wip-settings` cannot succeed. The AFT screen is ungated (`ui/src/app/settings.rs:231`).
- Inventory rows out of date in both directions: #290 "open, `test.fail()`" (L182) vs fixed and a plain test (`email-app.spec.ts:338-343`); delete identity "manual" (L123-124) vs `email-app.spec.ts:2382, 2407`; quarantine "manual" (L153, L243) vs L207; self-heal re-PUT "blocked" (L282) vs `live-node.spec.ts:2560`; Reply "asserts quoted body" (L189) vs asserts an empty body (L2250-2255); stale titles at L147, L150, L151, L165, L188.
- The suites list (inventory L21-66, QA skill L39-41) omits compose-autocomplete, migration-ui, mail-delegate-identity, export-permission and background-proof. 75 of 142 test titles appear nowhere in the inventory.
- The release skill omits the release freeze, the Cargo version check, `check-release-delegate.sh` and the pre-tag E2E; it mentions a "force" flag that `release.sh` does not have (L188-190) and memory files that are not in the repo (L162, L166, L210-213).

## 9. Suspected weak spots

1. Quarantine-by-skip can hide a real delivery failure in four cross-node tests (medium).
2. `grepLogDir` tails 256 KiB and is not scoped to the current test, so negative log assertions can miss or inherit lines (medium).
3. The archive half of the multi-round test is wrapped in `if ((await archiveBtn.count()) > 0)` (`live-node.spec.ts:1036-1049`): a missing button passes silently (high that the code allows it).
4. `TestIds` drift: a mistyped `TID.x` becomes `[data-testid="undefined"]` at run time (high on the fact).
5. `inbox-v2-live` has no CI and no `CI` guard, so revocation, RequireGhost against the real contract and the tab-closed drain are unguarded (high).
6. `repro-106-107` pump sends no `Origin` header, unlike every maintained pump (low-medium that it no longer answers prompts).
7. `inbox-v2-ghost.spec.ts:98-104` reads `data-seq` with no timeout (10 s wait when no outcome exists yet) and expects exactly `before + 1`; `inbox-v2.spec.ts:107-117` has the corrected form (low).
8. Both runners build a release bundle and then serve a dev build with `dx serve` (`Makefile.toml:973, 1001`; `build.yml:260-265`), so the release build is not what the tests exercise (medium).

## 10. Open work

- `test.fixme` delete + regenerate under the same alias (#5/#291), with a TODO to deliver a message first (`live-node.spec.ts:2258-2329, 2281`).
- Round 3 of the multi-round test is commented as blocked on #221 (L1017-1022), yet the #221 test exists (L1650).
- Sender-side cost avoidance under the verified-sender bypass awaits #85/#179 (L1517-1520).
- Gateway does not fan UpdateNotifications out to subscribed clients; "filed as a follow-up" (L1780-1788).
- Inbox v2 not covered anywhere: a renewal reaching a contact, the real ghostkeys delegate, the vault's return link through the gateway frame, the strangers-dropped count (inventory L56-63; `inbox-v2-live.spec.ts:33-36`).
- Inventory gap list L286-298 (nine items); blocked rows: read receipts #69, unarchive #60, AFT slot exhaustion #85.
- Testids "until we wire a build-time generator" (`ui/src/testid.rs:11-12`).

User-visible behaviour with no automated test (from the specs, not only the inventory): search filter; threading on/off toggle (`fm-threading-enabled-toggle` unused); multi-recipient send; auto-sign; `verify_on_send` block; hide unsigned; drafts-in-inbox; density, serif, font size; custom relay URL; edit contact; reply from Archive; empty-recipient send; Sent Pending → Failed; peer down and WebSocket reconnect; deleting the active identity; three or more identities; identity repair and unfinished-delete controls (`fm-id-repair-*`, `fm-id-finish-delete`, unused); orphan move to a chosen target (`fm-orphan-target`); consent-waiting state (`fm-backup-waiting`); declining any node prompt, including Background "Not now"; the container loader's redirect; v1 inbox and AFT-record contract migration; AFT usage panel; anything on Firefox or WebKit; any real-node flow on a mobile viewport.

## 11. Questions the tree does not answer

- Whether the fork has Ubicloud runners; without them none of the Playwright workflows start.
- Which checks are required by branch protection (only `check-drift` is described as required).
- Whether the gateway requires the `Origin` header on `/permission/*` (decides whether `repro-106-107` still works).
- Whether upstream intends `inbox-v2-live` to stay manual, given its run time.
- How often the quarantine skip fires in CI.

---

## Appendix: the reader's structured summary

ui/tests/ is one Playwright project: 15 spec files, 142 tests, two Chromium projects (desktop and Pixel 5), no webServer. Nine specs run offline against `dx serve --features example-data,no-sync` (real mail delegate in the page). Five need the isolated two-node network from scripts/run-isolated-nodes.sh: live-node (16 tests, one fixme), legacy-migration, background-proof, inbox-v2-live, and a manual repro. production-liveness runs against a deployed gateway.

CI runs the offline suite and live-node plus legacy-migration on every PR, and background-proof on path-filtered PRs and nightly. inbox-v2-live is in no workflow. All Playwright workflows use ubicloud runners, which the fork may not have.

Real-node specs answer every node prompt by polling /permission/pending and POSTing {"index":0}. They assert on node log files and on UI console lines. Four cross-node tests turn a delivery timeout into a skip when a known freenet-core relay signature is in the logs.

The QA inventory and both skills are stale. The inventory marks automated behaviour as manual and the reverse, cites five renamed test titles, and omits five spec files; 75 of 142 test titles appear nowhere in it. The QA skill documents a `wip-settings` cargo feature that does not exist. AGENTS.md is wrong on browser profiles (five vs two), the dev-server port (8082 vs 8092), a removed send gate, and CI scope. The release skill omits the release freeze, which is in force (.release-freeze exists).

I ran nothing; every finding is from reading the tree.

### Key facts

- Playwright config: two projects only, chromium (Desktop Chrome) and mobile-chrome (Pixel 5); retries 2; test timeout 120 s; trace on first retry; no webServer (ui/tests/playwright.config.ts:18-45)
- Default baseURL is http://127.0.0.1:8092 unless FREENET_EMAIL_BASE_URL is set (ui/tests/playwright.config.ts:24); CI and cargo-make always set it, using 8082 or the first free port in 8082-8182 (.github/workflows/build.yml:286, Makefile.toml:978-995, 1035)
- Three real-node specs are excluded by testIgnore unless FREENET_LEGACY_SEEDED / FREENET_BACKGROUND_PROOF / FREENET_INBOX_V2_LIVE is 1 (ui/tests/playwright.config.ts:11-17); legacy-migration and background-proof throw under CI when they cannot run (ui/tests/legacy-migration.spec.ts:33-38, ui/tests/background-proof.spec.ts:250-253)
- Offline suite is nine specs, 121 tests, each run on both projects: email-app 74, threads 10, threads-storybook 4, compose-autocomplete 9, inbox-v2 9, inbox-v2-ghost 5, migration-ui 7, mail-delegate-identity 2, export-permission 1. Build is `dx serve --features example-data,no-sync --no-default-features` (Makefile.toml:1001); example-data turns on local-delegate and test-ghost-master (ui/Cargo.toml [features])
- Offline CI job is ui-playwright-tests in build.yml: every PR and push to main, Chromium only, port 8082, `npx playwright test` with no file filter; live-node, production-liveness and repro-106-107 load and self-skip (.github/workflows/build.yml:167-295; ui/tests/live-node.spec.ts:193-197)
- cargo make test-e2e-real-node wipes and starts the iso nodes on free ports, publishes with publish-email-iso, then runs `npx playwright test live-node.spec.ts --project=chromium` (Makefile.toml:1068-1131). e2e-real-node.yml runs it and then test-e2e-legacy-migration on tags, every pull_request, nightly 04:30 UTC and dispatch, 45 min timeout (.github/workflows/e2e-real-node.yml:8-22, 44, 164-183)
- cargo make test-e2e-inbox-v2 (inbox-v2-live.spec.ts, test-ghost-master build, up to about 25 min) is referenced by no workflow; only Makefile.toml:1133-1196 and scripts/check-release-scrub_test.sh:149-160 name it
- cargo make test-e2e-background builds with e2e-probe and runs background-proof.spec.ts (Makefile.toml:1198-1263); e2e-background.yml runs it on PRs touching the listed paths and nightly 05:30 UTC, not a required check (.github/workflows/e2e-background.yml:11-26)
- Permission pump: every 200 ms GET {origin}/permission/pending with an Origin header, then POST /permission/{nonce}/respond with {"index":0} for each unseen nonce; one pump per node; errors swallowed (ui/tests/live-node.spec.ts:2831-2865). It allows every prompt; no spec tests a denial on a real node
- Log assertions: grepLogDir reads only the last 256 KiB of each freenet.* file (ui/tests/live-node.spec.ts:2983-3011); logHasLineWith reads whole files (2959-2981). Hard asserts include `Module cache miss` plus the hash in published-contract/candidate/mail_delegate_code_hash (289-314) and no `secret not found` (323)
- Quarantine: when cross-node delivery times out and coreRelayBugDetected() matches with no contract-side error in the peer log, the test calls test.skip instead of failing; four tests do this (ui/tests/live-node.spec.ts:596-606, 893-903, 1203-1211, 2126-2134, 2948-2952)
- live-node.spec.ts runs with retries 0 and unique aliases; one full run creates 14 identities on the gateway and 9 on the peer, against a per-origin cap MAX_IDENTITIES = 16 (delegates/mail-delegate/src/store.rs:36; ui/tests/live-node.spec.ts:57-96, 1878-1880)
- Tests wait on DOM hooks the UI publishes: data-pending-saves on html (ui/src/pending_saves.rs:62), data-jobs-run under e2e-probe only (ui/src/jobs.rs:173), data-autosave (ui/src/app.rs:6599), data-seq and data-delivered (ui/src/app/v2_view.rs:905-906)
- Offline fixtures are URL and storage hooks: ?story=, ?migration-fixture=not-installed|silent|data (ui/src/app/migration_fixture.rs:21-27), ?v2-foreign-fixture and ?v2-queue-fixture (ui/src/v2/owner.rs:172-180), localStorage fmail-ghostkeys-mock (ui/src/ghostkeys.rs:826)
- Testids: ui/branding/testids.json is read by ui/tests/testids.ts:169-171 and pinned against ui/src/testid.rs by the Rust test json_matches_rust_consts (ui/src/testid.rs:246). The TypeScript interface is unpinned and stale: 15 JSON keys missing, one phantom member fmArchiveUnarchive (ui/tests/testids.ts:55); there is no tsconfig
- scripts/check-release-scrub_test.sh:128-171 greps exact lines of the test-e2e-* tasks and requires FREENET_TEST_GHOST_ISO=1 and FREENET_E2E_PROBE_ISO=1 to appear in exactly one file among Makefile.toml, .github/ and scripts/release.sh; it runs in build.yml:165
- inbox-v2-live requires the tab-closed drain to come from the inbox notification within 120 s and the peer log to say woken_by=notification; otherwise it waits until 1,100,000 ms after the send for rescan and then fails (ui/tests/inbox-v2-live.spec.ts:45-47, 376-412)
- production-liveness.spec.ts is one test (brand, Create link, italic h1, clean console) run by scripts/smoke-test-production.sh:85 on both Chromium projects; it skips unless the base URL contains /v1/contract/web/ (ui/tests/production-liveness.spec.ts:35-41)
- Releases are frozen: .release-freeze exists and scripts/release.sh:37 refuses first; workspace version is 0.2.7 (Cargo.toml:56). release.sh runs cargo make test-e2e-real-node before tagging unless FREENET_RELEASE_E2E=0 (scripts/release.sh:226-233)
- Every workflow that runs Playwright uses runs-on: ubicloud-standard-8 (.github/workflows/build.yml:30, 168; e2e-real-node.yml:43; e2e-background.yml:45)

### Docs or comments that disagree with the code

- **[medium]** Claim (AGENTS.md:1010-1012; also .claude/skills/freenet-mail-release/SKILL.md:130, scripts/smoke-test-production.sh:20, RELEASING.md:202): The Playwright suite runs across 5 browser/viewport profiles: Desktop Chrome, Firefox, Safari, Pixel 5, iPhone 13 **Reality:** The config defines two projects, chromium (Desktop Chrome) and mobile-chrome (Pixel 5). CI installs Chromium only. Evidence: ui/tests/playwright.config.ts:31-43
- **[low]** Claim (code comment, ui/tests/playwright.config.ts:41-42): Run locally with: npx playwright test --project=firefox --project=webkit **Reality:** No firefox or webkit project is defined, so that command fails on an unknown project. Evidence: ui/tests/playwright.config.ts:31-40
- **[medium]** Claim (AGENTS.md:1028-1034): Manual run: start `dx serve --port 8082 ...`, then `cd ui/tests && npx playwright test` **Reality:** The config default baseURL is port 8092, so the recipe as written connects to nothing unless FREENET_EMAIL_BASE_URL is set. The spec headers say 8092. Evidence: ui/tests/playwright.config.ts:24
- **[medium]** Claim (AGENTS.md:1068-1070; docs/qa/manual-test-inventory.md:28-29; .claude/skills/freenet-mail-qa/SKILL.md:51): The send round trip in live-node.spec.ts is gated on FREENET_LIVE_E2E_SEND=1 **Reality:** No code reads that variable; the spec comments say the gate was removed and the cross-node tests always run. Evidence: ui/tests/live-node.spec.ts:445-448
- **[low]** Claim (AGENTS.md:1086-1088): Log assertions grep the gateway log for UPDATE_PROPAGATION and allocate_token **Reality:** Neither is asserted. The spec says current core never emits UPDATE_PROPAGATION; it polls other markers as a non-fatal diagnostic. allocate_token is not referenced in ui/tests. Evidence: ui/tests/live-node.spec.ts:547-563
- **[medium]** Claim (AGENTS.md:1093-1094; .claude/skills/freenet-mail-qa/SKILL.md:55-56; header comment .github/workflows/e2e-real-node.yml:5-6): The real-node E2E is intended for tags / nightly, not every PR **Reality:** The workflow triggers on every pull_request with no path filter, and its own later comment says so. Evidence: .github/workflows/e2e-real-node.yml:12-18
- **[medium]** Claim (docs/qa/manual-test-inventory.md:182): #290 is an open bug; the toast test is wrapped in test.fail() as a red guard **Reality:** The spec says #290 is fixed and the test is a plain passing test; the timer uses spawn_forever. Evidence: ui/tests/email-app.spec.ts:338-343
- **[medium]** Claim (docs/qa/manual-test-inventory.md:123-124): Delete identity (confirm modal, wrong-alias refusal) is manual only **Reality:** Both are automated offline, and a third test proves the delete reaches the delegate. Evidence: ui/tests/email-app.spec.ts:2382
- **[low]** Claim (docs/qa/manual-test-inventory.md:153, 243): Quarantine unknown sender and the quarantine_unknown toggle are manual only **Reality:** The toggle-on path (folder appears, badge count, rows diverted) is automated; only toggle-off is untested. Evidence: ui/tests/email-app.spec.ts:207
- **[medium]** Claim (docs/qa/manual-test-inventory.md:282): Own-inbox self-heal re-PUT after NotFound is blocked on iso because the branch cannot be triggered **Reality:** A live-node test triggers it by reloading at the reveal step and asserts the self-heal and its acknowledgement for both the inbox and the AFT record. Evidence: ui/tests/live-node.spec.ts:2560
- **[medium]** Claim (docs/qa/manual-test-inventory.md:147, 150, 151, 165, 188): Auto rows link to tests titled `opens an email and shows content`, `detail header shows full timestamp`, `detail header shows verified badge for incoming message`, `collapses a back-and-forth into one row ...`, `Forward prefills ... + quoted body` **Reality:** None of these titles exists. Current titles: `opens a conversation and shows the threaded detail`, `thread row shows a per-message timestamp`, `thread row shows a per-message trust pill ...`, `mixed list: one collapsed conversation row ...` (threads.spec.ts:121), `Forward ... + verbatim body`. Evidence: ui/tests/email-app.spec.ts:126
- **[low]** Claim (docs/qa/manual-test-inventory.md:189): Reply prefill test asserts the quoted body (since #274B) **Reality:** The test asserts the reply body is empty and has no `> ` quote; Forward keeps the body verbatim. Evidence: ui/tests/email-app.spec.ts:2250-2255
- **[high]** Claim (docs/qa/manual-test-inventory.md:232-236; .claude/skills/freenet-mail-qa/SKILL.md:58-83): WIP settings (AFT screen, read_receipts, pad_length, display_name) are gated behind the `wip-settings` cargo feature; verify with `cargo check -p freenet-email-ui --features wip-settings` **Reality:** ui/Cargo.toml has no wip-settings feature and no Rust source mentions it. The AFT screen is in the nav unconditionally and live-node drives it on the default build. Evidence: ui/src/app/settings.rs:231
- **[medium]** Claim (docs/qa/manual-test-inventory.md:21-66; .claude/skills/freenet-mail-qa/SKILL.md:39-41): Suites are email-app + threads (offline), live-node (iso), inbox-v2 trio, production-liveness, rust **Reality:** Five more spec files exist and are not listed as suites: compose-autocomplete, migration-ui, mail-delegate-identity, export-permission, background-proof. 75 of 142 test titles appear nowhere in the inventory. Evidence: ui/tests/migration-ui.spec.ts:26
- **[low]** Claim (docs/qa/manual-test-inventory.md:172; code comment ui/tests/threads.spec.ts:427-428): Rust tests `groups_legacy_by_heuristic_subject_and_participant` and `legacy_two_party_back_and_forth_splits_by_sender` exist **Reality:** Neither name exists. The tests are now `groups_legacy_by_heuristic_subject_across_senders` and `legacy_two_party_back_and_forth_folds_across_senders`. Evidence: ui/src/app.rs:2341
- **[low]** Claim (docs/qa/manual-test-inventory.md:202): repro-106-107.spec.ts asserts the draft persists, not sent-vs-draft routing **Reality:** It soft-asserts the sent message is NOT in Drafts. The in-session routing test `send routes the message into Sent, not Drafts (#291)` (email-app.spec.ts:928) is not cited. Evidence: ui/tests/repro-106-107.spec.ts:339-340
- **[low]** Claim (code comment, ui/tests/testids.ts:4-11): testids.ts is the single source of truth shared verbatim with the Rust UI **Reality:** The JSON is shared, but the TestIds interface lacks 15 JSON keys (four of them used by five specs) and declares fmArchiveUnarchive, which is not in the JSON. Nothing type-checks it. Evidence: ui/tests/testids.ts:55
- **[low]** Claim (code comments, ui/tests/email-app.spec.ts:832-835, 992-995, 2378-2380): Offline Drafts talk to a stubbed delegate path; CreateIdentity is dropped in offline mode; delete round-trips through the identity-management delegate **Reality:** The offline build links the real mail delegate into the page (local-delegate) and offline tests create, back up and delete identities through it. Evidence: ui/tests/mail-delegate-identity.spec.ts:5-9
- **[low]** Claim (scripts/smoke-test-production.sh:33-35, 89-93; RELEASING.md:224-230): `cargo make test-ui-live` runs live-node.spec.ts against a fresh `freenet local` node; the live-node test is 'landing in Phase 5 PR B' **Reality:** No such task exists. The task is test-e2e-real-node, and it uses two `freenet network` nodes in an isolated network. Evidence: Makefile.toml:1068
- **[high]** Claim (.claude/skills/freenet-mail-release/SKILL.md:24-28, 50-78): Release preflight is: clean tree, tag free, production key, GNU tar, node reachable, tools, frozen artifacts **Reality:** release.sh also refuses while .release-freeze exists (it does now), requires the workspace version to equal the release version, runs check-release-delegate.sh, requires fdev --as-state, and runs cargo make test, test-inbox, clippy and test-e2e-real-node before the first prompt. Evidence: scripts/release.sh:37
- **[low]** Claim (.claude/skills/freenet-mail-release/SKILL.md:188-190, 162, 166, 210-213): release.sh refuses a dirty tree or non-main branch 'but if forced via a flag' the tag is unrecoverable; see memory files fdev_update_needs_as_state.md, facade_state_framing.md, test_vs_prod_snapshot.md **Reality:** release.sh has no flag that bypasses the branch or clean-tree checks (only --yes and --check-freeze). The memory files are not in the repository. Evidence: scripts/release.sh:88-94
- **[low]** Claim (AGENTS.md ('cargo make test-e2e-inbox-v2' bullet); Makefile.toml:1134): When the notification does not drain, the inbox v2 run waits up to 18.3 more minutes for the rescan wake-up **Reality:** The fallback loop ends 1,100,000 ms (18.3 min) after the send, so it is at most about 16.3 minutes beyond the 2-minute bound. Evidence: ui/tests/inbox-v2-live.spec.ts:387
- **[low]** Claim (Makefile.toml:1199; AGENTS.md (Page fallback bullet)): test-e2e-background proves wake-up, import with the tab closed, cost, and page stall; the probe publishes probe_ms **Reality:** probe_ms is declared in the record type but no assertion or note reads it; nothing in the spec checks a page stall. Evidence: ui/tests/background-proof.spec.ts:64

### Suspected issues (leads from reading, not confirmed)

- **Quarantine-by-skip can hide a real cross-node delivery regression** (confidence: medium). Four live-node tests convert a delivery timeout into test.skip when a broad core-relay regex (including `missing contract parameters`) matches either node's log and a narrower contract-error regex does not. The spec itself notes the core signature also appears on healthy runs. A regression that stops delivery without one of the listed contract errors would show as skipped, and the workflow stays green. Evidence: ui/tests/live-node.spec.ts:596-606
- **Log greps are tail-limited and not scoped to the current test** (confidence: medium). grepLogDir reads only the last 256 KiB of each log file and matches lines written by earlier tests in the same run. Negative assertions (no `secret not found`, no panic, no `Rejecting SUBSCRIBE`) can miss a line that has scrolled out, or fail on a line an earlier test produced. Evidence: ui/tests/live-node.spec.ts:2994-3005
- **Archive assertions in the multi-round test are conditional** (confidence: high). The archive step runs only `if ((await archiveBtn.count()) > 0)`, and the preceding wait for the thread container has `.catch(() => {})`. If the archive button is not rendered, the test passes without checking archive at all, while the inventory counts this test as archive and read coverage on a real node. Evidence: ui/tests/live-node.spec.ts:1032-1049
- **Inbox v2 real-node path has no CI and no CI guard** (confidence: high). inbox-v2-live.spec.ts is run only by cargo make test-e2e-inbox-v2, which no workflow calls. Unlike the other two gated specs it only skips when it cannot run. Revocation, RequireGhost against the real contract and the tab-closed drain can regress without any check failing. Evidence: ui/tests/inbox-v2-live.spec.ts:244
- **TestIds interface drift removes any compile-time safety** (confidence: high). With no tsconfig and a stale interface, a mistyped TID key evaluates to undefined and the selector becomes [data-testid="undefined"], failing only as a timeout at run time. Five specs already use four keys the interface does not declare. Evidence: ui/tests/testids.ts:12-167
- **repro-106-107 pump omits the Origin header** (confidence: low). Every maintained pump sends Origin on both the GET and the POST; the repro spec sends neither and ignores the POST result. If the gateway checks Origin on /permission/*/respond, this spec never answers a prompt and its sends stall. Not verifiable without freenet-core source. Evidence: ui/tests/repro-106-107.spec.ts:36-45
- **inbox-v2-ghost clickSend is slower and stricter than the fixed version** (confidence: medium). It reads data-seq with no timeout, so when no outcome element exists yet the read waits the 10 s action timeout before falling back to 0, then requires exactly before+1. inbox-v2.spec.ts uses a 1 s timeout and a greater-than poll, with a comment explaining why. Evidence: ui/tests/inbox-v2-ghost.spec.ts:98-104
- **Offline runs build a release bundle that the tests never use** (confidence: medium). test-ui-playwright depends on build-ui-example-no-sync (a release dx build) and then starts `dx serve` without --release, which serves a dev build; the CI job does the same. A spec comment refers to the dev build's 84 MB WASM. The suite therefore never exercises an optimised offline build and pays for two builds. Evidence: Makefile.toml:973
- **live-node is two identities away from the delegate's per-origin cap** (confidence: medium). A full run creates 14 identities on the gateway under one origin; MAX_IDENTITIES is 16 and retries are off. Enabling the fixme test or adding two more gateway-side tests would make identity creation fail late in the run. Evidence: delegates/mail-delegate/src/store.rs:36
- **Reply-in-thread test's badge assertion cannot fail on a re-parented reply** (confidence: medium). The final check is count >= 3, the seeded size, which holds whether or not the reply joined the thread. Only the unchanged top-level group count guards the behaviour. Evidence: ui/tests/threads.spec.ts:414-420

### Open work

- test.fixme: delete + regenerate identity under the same alias shows no old messages (#5/#291); bleed mechanism undecided, TODO to deliver a message first (ui/tests/live-node.spec.ts:2258-2329, 2281)
- Round 3 of the multi-round test is commented as uncoverable until #221, though the #221 test now exists at line 1650 (ui/tests/live-node.spec.ts:1017-1022)
- Sender-side cost avoidance under the verified-sender bypass awaits AFT cap configurability #85/#179 (ui/tests/live-node.spec.ts:1517-1520)
- Gateway does not fan UpdateNotifications out to subscribed clients after CreateContact; the test reloads instead; 'filed as a follow-up' (ui/tests/live-node.spec.ts:1780-1788)
- Inbox v2 behaviour covered nowhere: a renewal reaching a contact, the real ghostkeys delegate on a node, the vault return link through the gateway frame, the strangers-dropped count past 1024 (docs/qa/manual-test-inventory.md:56-63; ui/tests/inbox-v2-live.spec.ts:33-36)
- Inventory gap list of nine items to automate: multi-recipient, verify_on_send toast, search filter, auto-sign on Resend, settings round trip, backup/restore round trip, 3+ identities, reply from Archive, Sent Pending→Failed (docs/qa/manual-test-inventory.md:286-298)
- Blocked rows: read receipts #69, true unarchive #60, AFT slot exhaustion #85 (docs/qa/manual-test-inventory.md:217, 228, 283)
- Firefox and WebKit are not configured or run anywhere (ui/tests/playwright.config.ts:41-42)
- Testid JSON and Rust consts are kept in sync by hand 'until we wire a build-time generator' (ui/src/testid.rs:9-13)
- 22 testids are referenced by no spec, including fm-threading-enabled-toggle, fm-id-repair-storage, fm-id-repair-list, fm-id-finish-delete, fm-orphan-target, fm-backup-waiting, fm-v2-ghost-recheck, fm-v2-queue-dropped (ui/branding/testids.json)
- No real-node test declines a node prompt; every pump answers index 0 (ui/tests/live-node.spec.ts:2849)
- Releases are frozen until the v0.3.0 chain is complete (.release-freeze:1; scripts/release.sh:37)

### Open questions

- Does the fork have Ubicloud runners? Every Playwright workflow uses runs-on: ubicloud-standard-8; without them the jobs never start.
- Which checks does branch protection require? The tree only describes check-drift as required; it does not say whether the offline Playwright job or e2e-real-node gate a merge.
- Does the freenet-core gateway require the Origin header on /permission/pending and /permission/{nonce}/respond? This decides whether repro-106-107 still works.
- Is inbox-v2-live meant to stay manual because of its run time, or was adding it to CI simply not done yet?
- How often does the core-relay quarantine skip fire in CI? The tree gives no record of skipped runs.
- Should the inventory be treated as authoritative before planning? As it stands, its auto/manual status is wrong in both directions for several rows.
- Does the offline suite need to pass against a release build? Both runners serve a dev build.

### Coverage

Read in full: `ui/tests/playwright.config.ts`, `ui/tests/package.json`, `ui/tests/testids.ts`, `ui/tests/app-name.ts`, `ui/tests/email-app.spec.ts`, `ui/tests/threads.spec.ts`, `ui/tests/threads-storybook.spec.ts`, `ui/tests/compose-autocomplete.spec.ts`, `ui/tests/inbox-v2.spec.ts`, `ui/tests/inbox-v2-ghost.spec.ts`, `ui/tests/inbox-v2-live.spec.ts`, `ui/tests/migration-ui.spec.ts`, `ui/tests/mail-delegate-identity.spec.ts`, `ui/tests/export-permission.spec.ts`, `ui/tests/live-node.spec.ts`, `ui/tests/legacy-migration.spec.ts`, `ui/tests/background-proof.spec.ts`, `ui/tests/repro-106-107.spec.ts`, `ui/tests/production-liveness.spec.ts`, `docs/qa/manual-test-inventory.md`, `.claude/skills/freenet-mail-qa/SKILL.md`, `.claude/skills/freenet-mail-release/SKILL.md`, `ui/branding/testids.json`, `ui/branding/app.json`, `.github/workflows/e2e-real-node.yml`, `.github/workflows/e2e-background.yml`, `scripts/smoke-test-production.sh`, `scripts/run-isolated-nodes.sh`

Skimmed: `Makefile.toml (lines 1-28, 512-661, 774-833, 900-1337)`, `.github/workflows/build.yml (lines 1-35, 100-295)`, `scripts/release.sh (lines 1-240, prompts by grep)`, `scripts/check-release-scrub_test.sh (lines 100-180)`, `scripts/select-guards.sh (lines 20-48)`, `scripts/select-guards_test.sh (lines 36-52)`, `ui/src/testid.rs (header and test module)`, `ui/Cargo.toml ([features] only)`, `ui/src/app/settings.rs (nav block, grep for gating)`, `ui/src/app/migration_fixture.rs (lines 1-30)`, `ui/src/pending_saves.rs (lines 1-30)`, `ui/src/jobs.rs, ui/src/toast.rs, ui/src/app.rs, ui/src/app/v2_view.rs, ui/src/app/login.rs, ui/src/v2/owner.rs (grep for test hooks and test names only)`, `RELEASING.md (headings and lines 25-36, 180-240)`, `AGENTS.md (lines 1006-1108 re-read for line numbers)`, `.release-freeze`, `ui/tests/package-lock.json (version lines only)`

Skipped: `ui/tests/package-lock.json body: lockfile, only the resolved Playwright version was needed`, `.github/workflows/check-contract-wasm.yml, inbox-v2-gates.yml, inbox-v2-laws.yml: no Playwright; grepped for runners and path filters only`, `tools/v1-node-seed/: the legacy-migration seed tool, outside my paths; only its invocation was read`, `scripts/check-release-scrub.sh: read only through its test and the tasks that call it`
