---
name: desktop-setup
description: Set up an Electron desktop app's foundation once, before the first screen is designed or built — the three-process scaffold with the typed IPC bridge, app identity per environment, the session and the gateway in Main, the data layer, mock mode, the design tokens, windows, menus, deep links and updates, the test harness, the signed build pipelines, and the docs sync with the backend repo. Use at the start of a desktop client project, or to bring an existing one up to the foundation the build skills assume.
---

# Desktop Setup

Builds the foundation every screen stands on. It runs once per project, before the first `/desktop-build`. It needs no product docs and no design system, so it can run while those are being written — only the design tokens step waits for `DESIGN.md`, and it can be run again later for that step alone.

It builds no screens: not even sign-in, which is a feature area designed and built from its own docs. It adds only what every desktop app needs. A library only some features use (tables, charts, a local cache, a terminal, folder watching) is added by the first build that needs it.

## Read first

- **The rules in `.claude/rules/`:** `tech-stack`, `coding-standards`, `security`, `testing` and `ci-cd` (and `analytics` where installed). They define what this skill builds, and it does not restate them.
- **`CLAUDE.md`:** the audiences and clients, which audience this repo serves, and the project configuration. If the backend repo path, the app id or the staging API URL is not recorded, ask once.
- **The backend's `docs/api-docs/`** if any exist yet, for the auth endpoints' exact paths and the platform and version headers. Until they do, the auth paths are constants in `main/session/paths.ts`, set to the backend's conventional paths and corrected when the auth area is integrated; the headers are set in `main/gateway/fetch.ts`.

If the repository already has code, take stock first. Keep what meets the rules, add only what is missing, and list anything that conflicts with the rules for me to decide. Never rebuild working pieces unasked. A renderer with Node integration, a disabled sandbox or a remote URL loaded in a window is the first thing to raise: the security rule does not bend for existing code.

## Step 1 — Plan → **wait for my approval**

State what exists, what will be created, and anything the rules leave to this project: the app id and scheme, the platforms, which sign-in providers, whether the app has a tray or runs in the background, file associations, store distribution. Check the current stable version of Electron and each tool in `tech-stack` rather than relying on memory — Electron ships a major every eight weeks and electron-builder's signing options move with the platforms — and say which versions the scaffold will use. Check the current names of the settings and options the rules name (pnpm's supply-chain settings, electron-builder's signing and publish options, Electron's Fuses) against each tool's current docs: they change between major versions.

## Step 2 — Scaffold the application

- **The project per `tech-stack`:** electron-vite with TypeScript strict and three `tsconfig` projects, React with React Compiler on, Tailwind, ESLint and Prettier configured as `coding-standards` says, a pre-commit hook running lint-staged, pnpm with its settings in `pnpm-workspace.yaml` (the supply-chain ones `security` requires, `allowBuilds` for Electron and the native modules), `@electron/rebuild`, a committed lockfile, and `.env` and `.env.*` gitignored except `.env.example`.
- **`electron-builder.yml` keyed by `APP_ENV`:** app id with `.dev` and `.staging` suffixes, the display name with its suffix, the icon variants, the scheme, the feed URL, the targets per platform, the macOS entitlements for the hardened runtime, notarization, Azure Trusted Signing, the Fuses `security` lists, ASAR integrity, and the staging prerelease version. A `beforePack` hook it references fails a production build with mock mode on.
- **`electron.vite.config.ts`** injecting the environment's constants (API URL, feed URL, Sentry DSN, mode) through `define`, declared and validated in `src/shared/env.ts`.
- **The three processes** from `coding-standards` → Project structure: `src/main/` with its capability modules and `register()` functions, `src/preload/index.ts` exposing `window.api` from the contract and nothing else, `src/renderer/` with the folder layout, a placeholder `features/.gitkeep`, and `src/shared/` with the IPC contract, the environment and the shared types.
- **The IPC contract** in `src/shared/ipc.ts`: every channel `coding-standards` names, each with its Zod payload and result schemas; `main/ipc.ts` registering handlers with payload parsing and sender validation; `preload` building the bridge from it; `renderer/src/lib/api.ts` typing the renderer side.
- **Windows:** `main/windows/` creating the main window with the `BrowserWindow` options `security` requires (asserted by a test), the platform title-bar style, state persistence through `electron-store` clamped to a visible display, `ready-to-show`, and the secondary-window factory; the `app://` scheme registered as privileged and served by `protocol.handle` from the renderer build, with the CSP response header set on it and on the dev server; `will-navigate`, `setWindowOpenHandler` and `setPermissionRequestHandler` denying by default; devtools unbound in production.
- **shadcn/ui initialised** with the base components every product needs: button, input, label, form, select, checkbox, radio, switch, textarea, dialog, sheet, dropdown menu, popover, tooltip, toast (sonner), tabs, card, badge, avatar, skeleton, separator, table, command (the command palette's base). More are added per build.
- **The `components/shared/` pieces every product has:** the shell with its title area and drag region, sidebar, command palette, empty state, error state, offline indicator, confirm dialog, update banner, and a loading skeleton for lists and forms. The shell is only a placeholder here (a title area, a sidebar slot and a content area): its form is decided by `/design-system` and it is designed and built in `/desktop-build`'s Phase 0.
- **The router** in `renderer/src/app/router.tsx`: `createHashRouter` with the `(auth)` and `(app)` trees, each with a layout and an error boundary, a not-found route, the session guard from `useSession`, and the readiness guard for un-integrated units. Routes are placeholders that the builds replace.
- **Fonts** bundled from `assets/fonts/` and declared in `globals.css`; the theme following `nativeTheme` over the bridge, wired for dark mode.

## Step 3 — Session, gateway and data layer

Exactly as `coding-standards` → sections 3 to 5 describe:

- `main/session/`: the access token in memory, the refresh token in `safeStorage` (and the signed-out-per-launch behaviour when `safeStorage` is unavailable), launch and wake refresh, the PKCE sign-in flow through the system browser and the scheme callback, sign-in and sign-out with the clearing order `coding-standards` gives, and the `session:changed` event.
- `main/gateway/core.ts` with `ApiError`, the status table, and the base URL that mock mode points at the in-process dispatch; `main/gateway/fetch.ts` with the bearer header, the platform and version headers, single-flight refresh-and-retry, and the offline mapping; `main/gateway/realtime.ts` with the SSE and WebSocket forwarding.
- `renderer/src/lib/api.ts` rebuilding `ApiError` from the bridge, and `lib/query-client.ts` with the defaults (`staleTime`, retry rules), `onlineManager` from Main's online events and `focusManager` from window focus.
- `hooks/use-me.ts`, `hooks/use-session.ts`, `hooks/use-online.ts`, `hooks/use-update-state.ts` with its `supported` / `soft` / `force` states and the placeholder screens for the last two, `hooks/use-shortcut.ts` reading the shared shortcut table.
- `pnpm api:types`, generating `src/shared/api.ts` from the backend's OpenAPI spec recorded in `CLAUDE.md`. The backend exposes one only once it is built, so the script is wired here and first run at the first integration; until then every feature's types are provisional, per `coding-standards` → The bridge and the gateway.
- **Local dev against a deployed backend:** a `pnpm dev:staging` script that starts electron-vite with the API URL pointed at staging in direct mode. Bearer tokens work from any origin, and the gateway runs in Node, so no CORS or hosts entry is needed.

## Step 4 — Mock mode

The semantics — the mode switch, in-process dispatch in Main, passthrough, sessions, the sign-in-first rule, the production exclusion, the readiness gate — live in `coding-standards` → Mock mode and are not restated here. This step creates the pieces:

- The dispatch in the gateway's `core` behind `API_MODE`, importing the mock module only outside production.
- The handler helper: `defineHandlers({ 'GET /orders': ({ query, session }) => ... })`, with path parameters, latency simulation, in-memory state for the app's lifetime, and the `?mock=error` switch — plain functions with no Electron imports, so the renderer's test double runs them too.
- `main/mocks/auth.ts`, the mock auth for the conventional auth paths (`/auth/sign-in`, `/auth/sign-out`, `/auth/refresh`, `/auth/me` — corrected at integration), returning JWT-shaped tokens through `session`.
- `renderer/src/test/api-double.ts`, the `window.api` double over the same handlers for Vitest.
- One example feature folder, `features/_example/`, showing provisional types, a query hook, a virtualised infinite list, a mutation hook, fixtures, handlers, a screen with every state, a form, and a shortcut, deleted by the first real build.

## Step 5 — Design tokens *(when `DESIGN.md` exists; otherwise placeholders, and re-run this step later)*

- `styles/globals.css` defining `DESIGN.md`'s colors, radii, spacing and type scale as CSS variables under the shadcn names — each color in `:root`, its `-dark` twin under `.dark` — the shadows from its Elevation & Depth table, the minimum window size and the content width from its Layout section, and the panel breakpoints from its Window Sizes table, all exposed to Tailwind.
- The shadcn components adjusted to the design system's radii, borders, focus rings and type.
- The fonts `DESIGN.md` names, bundled.
- The durations and easings from its Motion table as CSS variables, and a `lib/motion.ts` exporting them for Motion.
- Main's `nativeTheme.themeSource` wired to the theme preference, so native menus and dialogs match.

## Step 6 — OS layer

Wired once, used by the builds:

- **The application menu** per platform from the shared shortcut table, with the standard roles (`undo`, `copy`, `paste`, `selectAll`, `minimize`, `close`, `quit`, `about`) and the Help menu's links through the external-link allowlist; native context menus through the bridge.
- **Deep links:** scheme registration, the single-instance lock handing arguments to the running instance, parsing and validation, the `state` check for sign-in callbacks, routing to a window, and the sign-in redirect that remembers a validated route. Files opened from the OS go through the same path.
- **Files:** `main/files/` with open and save dialogs, dropped paths, reads and writes under session-granted roots with traversal and symlink checks, and streamed transfers for anything large.
- **Updates:** `main/updates/` with `electron-updater` against the environment's feed, the check on launch and every few hours, background download, the update state over the bridge, install on quit or on request, and `useUpdateState`'s force-update screen against the backend's minimum-version endpoint (a configured path until the api-docs name it).
- **Tray and notifications**, where `CLAUDE.md` says the product has them: the tray with its menu and the close-to-tray behaviour; notifications through Main with click-to-route.
- **Close and quit** per platform, with the pending-work guard.
- **Sign-in providers** the plan chose: the PKCE flow against the backend's native OAuth endpoints, and the email-and-password form's gateway calls.

## Step 7 — Test harness

- Vitest with the `renderer` and `main` projects, `vitest.setup.main.ts` mocking `electron` once, `renderWithProviders` with the `window.api` double, and the gateway's MSW setup (`msw/node`) from the shared handlers with `onUnhandledRequest: 'error'`.
- Playwright configured with `_electron.launch` against `electron-builder --dir` output (under `xvfb-run` on Linux, since the Electron launcher has no headless mode), a clean `userData` per run, a helper for the dialogs, and the smoke project (3–5 specs, tagged) and the full project.
- `e2e/README.md` listing the journeys and why each is there; the smoke suite starts with "the app launches, paints the shell within the budget, and sign-in renders".
- One working test of each kind against the foundation: a unit test of a formatter and one of an IPC handler's validation, an integration test of the example screen, the gateway's refresh test against MSW, the `BrowserWindow` options assertion, a Playwright check that the packaged renderer carries the CSP, and the smoke spec on the unpacked build.

## Step 8 — Observability and analytics

- Sentry per `security`: `@sentry/electron` in Main and renderer, scrubbing, native crash reporting, source maps uploaded at package time, environment per build, the error boundaries reporting, expected `ApiError`s and offline failures filtered.
- `electron-log` in both processes with the scrubbing, rotation and levels `security` says.
- Where `analytics` is installed: the PostHog provider in the renderer, screen tracking from the router, identify and reset wired to the session events, the CSP entry, and `docs/analytics-events.md` started.

## Step 9 — Pipelines

- **The three GitHub Actions workflows `ci-cd` defines:** PR checks (three type-check projects, lint, tests, the unsigned packaging check, the `e2e` label path with the platform matrix), the staging deploy on every platform with signing, notarization, the R2 or S3 publish to the staging prefix, the `staging-current` tag, the Playwright run and the source-map upload, the `hotfix/*` triggers, and the production deploy with its release-owner check, SHA-only `commit` input, version check against the last `v*` tag, the pending prefix, the smoke run, the `stagingPercentage: 10` feed write, the source-map upload and the `v*` tag.
- **The feed scripts:** publishing to the bucket, writing and editing `stagingPercentage` in the feed files, promoting from pending to production, and writing `downloads.json` — the current version and the installer URL per platform and architecture — next to the feed for the download page to read.
- **Dependabot or Renovate** configuration, and the security policies `security` → Updates and supply chain requires. Check electron-builder's and Electron's current documentation for the signing, notarization and Fuses options; they change.

## Step 10 — Docs sync with the backend

- `scripts/docs-sync.mjs` with `pnpm docs:pull` and `pnpm docs:push`, reading the backend repo path from `CLAUDE.md` and keeping, per synced file, the hash of the version last pulled:
  - **pull** copies from the backend every doc `documentation-conventions` lists as arriving from it, plus the backend's `docs/screen-specs/` and phase screen-spec files where they exist, and `docs/integration-handoff/` where it exists, replacing this repo's copies, and reports what changed. A file an earlier pull brought that no longer exists in the backend is deleted here and reported — the backend's promotion deletes a feature's phase folder, and the canonical files are then the specs. It refuses to overwrite or delete a file with uncommitted local changes, and never replaces a committed screen-spec draft that was never pushed. **This client's handoff files are the one exception to replacing:** they are three-way merged against the last-pulled version, so entries committed here and resolutions appended there both survive — both sides only append, so the merge never conflicts, and a real conflict stops with a message.
  - **push `--specs {file}`** copies one screen-spec file (a canonical area file, or a phase feature's file) to the same path in the backend's `docs/`; **push `--handoff`** copies `backend-issues-desktop.md` and `backend-issues-desktop-resolved.md` and nothing else. Both first check that the backend's copy is the version this repo last pulled, and stop with a message to pull first if it is not, so a stale copy never overwrites the backend's edits. Push copies nothing else, and refuses to copy a file with uncommitted changes, so what the backend receives is always a committed version.
- The `docs/` folders from the documentation map, with `.gitkeep` files where nothing exists yet. `docs/feature-areas.md` is never created here; it arrives from the backend.

## Step 11 — Infra guide

Write `docs/infra/infra-setup.md`: every step only I can do, numbered, so staging and production can be set up start to finish by following it.

- **The update feed:** the R2 or S3 bucket, its public domain (`updates.{domain}`) with HTTPS, the `staging/`, `pending/` and production prefixes, and an access key limited to that bucket for the workflows.
- **On Apple:** the Developer Program, the Developer ID Application certificate exported for CI, the App Store Connect API key for notarization, and the Team ID for the hardened-runtime entitlements.
- **On Azure:** the Trusted Signing account and certificate profile, the app registration whose tenant, client id and secret the Windows job uses, and the publisher name as it will appear in the installer.
- **On GitHub:** branch protection with PR checks required, squash merges, the release owners (whose usernames go into the production workflow), the `staging` and `production` environments holding the signing identities, the notarization key, the Azure credentials, the bucket key and the Sentry token, deploying only from `main` and `hotfix/*`; secret scanning with push protection.
- **Elsewhere:** the Sentry project per environment, the download page on the product's web domain (served by the web client or the backend) reading its links from the feed's `downloads.json`, the stock photo API keys and the image generation account if `/desktop-build` will use them, and the analytics projects where installed.
- **With the backend:** confirm the backend contract from `security` with the backend developer, item by item, before the first integration — the platform header and tokens in the body, the native OAuth flow and this app's registered scheme, the minimum-version endpoint and the download page, account deletion.

Then **offer to walk me through it live.** Once the bucket and the signing identities exist, record the identifiers, the dev and test commands, the E2E command and the sync commands in `CLAUDE.md`.

## Step 12 — Verify and report

1. **Verify from a clean clone:** install; `pnpm dev` in mock mode opens the main window and renders the example screen with mock data; the guard sends a signed-out launch to the auth tree; the mock sign-in stores the refresh token in `safeStorage` and `useMe` resolves; a quit-and-relaunch restores the session through the refresh path; a deep link to the example screen opens it in the running instance; the window's bounds survive a restart; lint, the three type checks, unit and integration pass; `electron-builder --dir` succeeds for the current platform and the renderer bundle is under budget; the `BrowserWindow` options test passes; the smoke suite passes on the unpacked build; opening and closing a secondary window ten times leaks nothing.
2. **Verify mock mode's edges:** a path with no handler is forwarded to the configured API URL with the `Authorization` header; a build with `APP_ENV=production` ignores the mode and the test proves it.
3. **Report:** what was created, the commands, what remains for me (the infra guide), and any open decisions. The checks that need the signing identities and the feed — a signed build, notarization, an update from the feed — run once the infra guide's Apple, Azure and feed sections are done; until then list them as pending.
4. **Deliver it as a PR.** Create a branch off `origin/main`, push to it, and give me the link to create the PR. Never create the PR yourself. The PR checks workflow added here runs on that PR. Say plainly that the PR is merged only after the infra guide's feed, signing and GitHub sections are done: until then the staging workflow fails at its first signing step.
