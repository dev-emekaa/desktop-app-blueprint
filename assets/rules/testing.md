# Testing

How this desktop client is tested. The tools are named in `tech-stack`; how tests are set up in this stack is in `coding-standards`; which pipeline runs what is in `ci-cd`. What each build, integration or fix must cover is set by those skills.

## Principles

- **Test the way users use the app:** find elements by role, label or text, interact, assert on what is visible. Never on component internals, state or implementation details.
- **Mock at the boundary the process has.** The renderer's boundary is the bridge, so its tests replace `window.api` with a double backed by the shared mock handlers. Main's boundary is the network, so the gateway's tests use MSW. Never mock our own modules, hooks or child components.
- **Most confidence comes from integration tests.** A small E2E suite guards the critical journeys on the packaged app, on every platform. Production only ever receives a commit already verified on staging, and a release is checked on real machines before it rolls out, because an installed version cannot be recalled.

## Layers

| Layer | Proves | Share | Runs against | When |
| --- | --- | --- | --- | --- |
| **Static** | Types in all three processes, lint, formatting, bad test patterns, accessibility lint, the renderer imports nothing from Node, the packaging config (`electron-builder --dir`) | Always on | The source | Before commit, and in every pipeline |
| **Unit** | Pure logic: utils, formatters, Zod schemas, query-key builders, store reducers; Main's modules — the gateway, the session, the IPC handlers, deep-link and file-path validation | ~20–25% | Nothing; MSW for the gateway | Locally, on every PR and merge |
| **Integration** | A whole screen or feature rendered with its real children, providers, router and forms, over the `window.api` double | ~55–65% | jsdom | Locally, on every PR and merge |
| **E2E** | 5–20 critical journeys through the packaged app on macOS, Windows and Linux | ~10–15% | The packaged app | On PRs labelled `e2e`; the full suite on the staging artifacts; smoke on the production artifacts |
| **Manual QA** | The integrated flows a person should walk once, on real machines | — | The staging build | Before every production deploy, from a 10–15 minute checklist and the machine matrix |

Coverage is read to find untested logic, never enforced as a percentage.

**Deliberately not used:** snapshot tests of component trees; unit tests of presentational components with no logic; Spectron and WebdriverIO (Playwright's Electron launcher covers E2E); mocking our own modules.

## Environments and data

| | Unit | Integration | E2E | Manual QA |
| --- | --- | --- | --- | --- |
| **Backend** | None; MSW for the gateway | The `window.api` double over the shared handlers | PR artifacts: mock mode, passing through to staging · Staging artifacts: the real backend, on its seed | Staging, on its seed |
| **Session** | A fake `safeStorage` | A signed-in `me` through `renderWithProviders` | The mock auth until the sign-in unit is integrated; real sign-in with the seeded test users after | Seeded test users |
| **Data** | Built per test | Fixtures from `features/*/mocks/`, overridden per test through the double | The backend's seed | The backend's seed |
| **Machine** | — | jsdom · Node | GitHub's macOS, Windows and Linux runners | The machine matrix below |

- **Automated tests never touch production data.** Production smoke tests are read-only, with a dedicated low-privilege account, on the production artifacts.
- **E2E tests depend on the backend's seed.** They use the records the seed marks as disposable and clean up what they create; they never reseed staging on their own, because manual QA and the backend's own verification share it. A reseed is asked for, through the backend's reseed dev-tool endpoint, when the seed has drifted.
- **Integration tests start from the shared mock handlers** so a screen is tested against the same shapes it was built on; a test that needs a different response overrides the one handler it needs. Once a unit is integrated and its handlers are removed, its tests answer its endpoints from the updated fixtures through the same per-test overrides.
- **E2E runs use a clean `userData`** per run (`--user-data-dir` to a temp folder), so no run inherits another's session, preferences or cache.

## Unit tests

- **No network, file system, timers or randomness** unless faked. Schemas, formatters and key builders run for real; Main's modules run against a mocked `electron` (`vitest.setup.main.ts`), a fake `safeStorage` and a temp directory for `electron-store`.
- **The gateway** is tested against MSW with the fixtures: headers added, refresh single-flight, retry once, sign-out on a failed refresh, `ApiError` shapes, network failure mapped to offline.
- **Every IPC handler** is tested through the contract: a valid payload reaches the module, an invalid one is rejected before it does, a sender that is not one of the app's windows is refused.
- **Path validation** is tested with traversal and symlink cases; deep-link parsing with malformed links and a stale `state`.
- **Group by the function under test, one behaviour per test**, named for the behaviour: `rejects a path outside the granted root`.

## Integration tests

- **Render the screen through `renderWithProviders`**, with a fresh `QueryClient` per test, retries off, the router at the screen's route, and the `window.api` double answering from the fixtures.
- **Drive it as a user would**: `user-event`, not `fireEvent`; `findBy*` for what appears after a request; `getByRole` before `getByTestId`, and a test id only where no accessible query exists. Keyboard paths are tested too: Tab order, Enter, Escape, the screen's shortcuts.
- **Cover the screen's states**, not only the happy one: loading, empty, error, offline, the permission-denied state, and every conditional state the screen spec lists.
- **Forms:** submit valid data and assert the request the double received; submit invalid data and assert the field errors; return a 422 from the double and assert it maps onto the fields.
- **Bridge events** (a deep link, an update state, a session change) are fired through the double and the screen's response asserted.
- **Each test sets up its own double and leaves no shared state.** Tests pass in any order.

## E2E tests

- **Critical journeys only:** sign-in with email (the system-browser OAuth flow is in the device check, not in E2E), the primary flow of each feature area, checkout where there is one (opening the system browser is asserted, not followed), a deep link into a protected screen, a file open and save through the dialogs (Playwright's dialog handling), the update banner against a test feed, and anything that would make the product unusable if broken. The list is kept in `e2e/README.md` with the reason each journey is there.
- **Against the packaged app, never `electron-vite dev`:** Playwright's `_electron.launch` on the unpacked build (`electron-builder --dir`) for PR and staging runs, on the signed artifacts for smoke.
- **All three platforms:** every journey runs on the macOS, Windows and Linux runners (Linux under `xvfb-run`); a journey that needs a platform step (the menu bar, a shortcut modifier) branches on the platform rather than skipping one.
- **Stable by design:** role-based selectors, `expect` with auto-waiting, no fixed sleeps, a clean `userData` per run. A flaky test is fixed, not retried until it passes.

## Smoke tests

- **Three to five read-only flows** on the signed artifacts: the app launches and paints the shell within the startup budget, sign-in works, the main screens render their data, a deep link opens its screen, the update check completes against the feed. Until the sign-in unit is integrated the sign-in is the mock auth's; after, it is the smoke account against the real backend.
- **Run after every deploy** — on the staging artifacts after a staging deploy, and on the production artifacts before the rollout opens — and a failure stops the release as `ci-cd` says.

## Machine matrix

The manual checklist runs on, at least:

| Machine | Why |
| --- | --- |
| A Mac on Apple silicon, and one on Intel or Rosetta | The two macOS architectures; notarization and Gatekeeper on a clean install |
| Windows 11, with SmartScreen at its default | The signature's reputation, the NSIS installer, the title-bar overlay |
| Ubuntu (current LTS), with the AppImage and the `.deb` | The native frame, `safeStorage`'s secret-store dependency, file associations |

On each: a clean install and an update from the previous version; light and dark following the OS; 100% and 200% display scaling; the minimum window size and a maximised window on a large display; the keyboard-only pass of the primary flow; sleep and wake mid-session; airplane mode mid-flow.

## When something breaks

- **A bug found anywhere gets a test that reproduces it**, at the lowest layer that can (usually integration; a Main bug at unit). The test fails before the fix, passes after, and ships in the same PR.
- **A bug that manual QA catches twice becomes an automated test.**
- **A crash in Sentry gets a test when it can be reproduced**, and a note on the release when it cannot; a native crash is reproduced on the platform it came from before it is called fixed.
- **A failing test is never skipped, deleted or loosened** to turn a pipeline green unless the PR states why.
