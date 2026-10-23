# Coding Standards — Electron · React · TypeScript

How code is written in this desktop client. The tools are named in `tech-stack`; security policy is in `security`; what tests must prove is in `testing`. Where a rule here touches those, it gives the mechanism in this stack and leaves the policy to them.

## 1. Architecture

Three processes, one direction of trust. The renderer draws. Main touches the operating system and the network. The backend decides everything else.

| Process | Owns | Never |
| --- | --- | --- |
| **Main** (Node) | Windows, menus, tray, dialogs, files, deep links, updates, the session and its tokens, every API call (the gateway), mock mode, logging | Business rules, UI, data the backend owns |
| **Preload** | The bridge: exposes `window.api` from the IPC contract, and nothing else | Logic |
| **Renderer** (Chromium, sandboxed) | React, routing, TanStack Query, forms, UI state | Node, the file system, tokens, network calls of its own |
| **Backend** | Data, validation, business rules, authentication, authorization, issuing and rotating tokens, entitlements | — |

Business rules are enforced by the backend. The app repeats a rule only where the user would otherwise wait for a round trip to learn it (a form field's limit, a disabled action), and that repetition is a convenience, never the enforcement.

### Data flow

One path. A screen renders a query hook → the hook calls `window.api.fetch` → preload forwards it over IPC → Main's `gateway` adds the bearer header and calls the backend (or dispatches to a mock handler) → the response returns the same way → the cache updates and the screen re-renders.

**Rules:**

1. Default `staleTime` above zero (60 seconds unless an entity says otherwise). Refetch on window focus is on.
2. One `QueryClient` for the app's lifetime, in `lib/query-client.ts`, with `onlineManager` fed by Main's online events and `focusManager` by window focus.
3. Query keys are defined once per entity, in the feature's `api/keys.ts`, and used by every hook.
4. Mutations go through TanStack Query to the gateway, then invalidate the affected keys.
5. No `useEffect` to fetch. Data comes from a query hook.
6. **Offline is a state, not an error.** A query with cached data shows it with the offline indicator; a query with none shows the offline state; a mutation fails fast with a retry toast. An offline queue for mutations, or a local cache that works offline, is a product decision made in a build plan, never a default.
7. **The bridge carries requests, not data models.** `window.api` exposes a small, stable set of capabilities; it never grows a method per screen.

## 2. Project structure

```
electron.vite.config.ts         the three builds; define() injects the environment constants
electron-builder.yml            packaging, signing, feeds, per APP_ENV
e2e/                            Playwright specs against the packaged app, and README.md listing the journeys
assets/                         icons per platform, fonts, dmg background
src/
  shared/                       imported by every process; no runtime dependencies on either side
    ipc.ts                      the IPC contract: channel names, Zod schemas for every payload and result
    api.ts                      generated from OpenAPI — never edited
    env.ts                      the build-time constants, validated
    types.ts                    types both sides use (session state, update state)
  main/
    index.ts                    app lifecycle: single instance, ready, activate, quit; wires the modules
    windows/                    BrowserWindow creation, state persistence, the main and secondary windows
    menu/                       the application menu and context menus, per platform
    tray/
    deep-links/                 scheme registration, argument parsing and validation, routing to a window
    session/                    the access token in memory, the refresh token in safeStorage, sign-in and sign-out
    gateway/
      core.ts                   base URL, mock dispatch in mock mode, JSON, ApiError
      fetch.ts                  bearer header, platform and version headers, single-flight refresh-and-retry
      realtime.ts               SSE and WebSocket connections, forwarded over IPC
    files/                      dialogs, drops, reads and writes under validated paths
    updates/                    electron-updater: check, download, install on quit, the update state
    notifications/
    store.ts                    electron-store: preferences, window state
    mocks/
      index.ts                  collects every feature's handlers; the gateway reads it outside production
      auth.ts                   mock sign-in, sign-out, refresh and me, until the auth area's own mocks replace them
    ipc.ts                      registers every handler from the contract, validating payloads and senders
    logger.ts                   electron-log, scrubbed
  preload/
    index.ts                    contextBridge.exposeInMainWorld('api', …) from the contract — nothing else
  renderer/
    index.html
    src/
      app/                      routes only: the router, layouts, thin route files
        router.tsx              createHashRouter; the (auth) and (app) trees; the readiness and session guards
        layouts/                the shell layout, the auth layout
      features/
        <feature-area>/         one folder per feature area, named after docs/feature-areas.md
          api/
            keys.ts             query keys
            types.ts            provisional types while the area is mocked; deleted at integration
            queries.ts          useX() hooks
            mutations.ts        useXMutation() hooks
          components/           screens and parts that belong to this area only
          schemas/              Zod schemas for this area's forms
          mocks/
            fixtures.ts         typed sample data
            handlers.ts         mock handlers: method + path → response (run in Main; reused by tests)
          store.ts              Zustand store, only if the area has client-only state
          index.ts              what other code may import from this area, and the area's readiness
      components/
        ui/                     shadcn components, owned and edited here
        shared/                 product-wide pieces: shell, title bar, sidebar, command palette, empty state, error state, offline indicator, confirm dialog, update banner
      lib/
        api.ts                  the renderer's typed wrapper over window.api.fetch; ApiError
        query-client.ts
        utils.ts                cn() and small pure helpers
      hooks/                    product-wide hooks (useMe, useOnline, useUpdateState, useShortcut)
      styles/
        globals.css             Tailwind theme: the DESIGN.md tokens as CSS variables
```

**Dependency direction, renderer:** `lib` ← `components` ← `hooks` ← `features` ← `app`. Nothing imports upward, and a feature never imports another feature's internals: cross-feature use goes through the other feature's `index.ts`, and a component needed by two features moves to `components/shared/`. **Across processes:** `main`, `preload` and `renderer` import from `shared` and never from each other. The renderer imports nothing from Node; the build fails if it does.

**Routes stay thin.** A route file renders a screen from a feature and nothing else. Screens, hooks and schemas live in `features/`. A screen that belongs to several areas lives in the area whose entity's lifecycle it advances, as the screen specs decide. Folders are always feature areas: a later phase's feature puts each of its screens in the area it belongs to, never in a folder named after the feature.

**Main is organised by capability**, one module per folder above, each exporting a `register()` that adds its IPC handlers from the contract. A module never reaches into another's state; they talk through the contract's events.

## 3. The bridge and the gateway

**The IPC contract** (`shared/ipc.ts`) is the only place a channel is named. Each entry has a channel name, a Zod schema for the payload, and a Zod schema for the result. Preload builds `window.api` from it; Main's `ipc.ts` registers a handler for each, parses the payload with its schema before the handler runs, and checks the sender is one of this app's windows. A channel that is not in the contract does not exist.

**`window.api` is small:** `fetch`, `realtime.subscribe`, `session.*` (status, signIn, signOut), `files.*` (open, save, read, write — under dialog-granted paths), `window.*` (minimize, maximize, close, titleBarActions), `shell.openExternal`, `updates.*` (state, check, install), `prefs.*` (get, set), `clipboard.write`, `notifications.show`, `menu.popup(items)` for native context menus (returning the chosen item's id), `platform` (a constant), and `on(event, handler)` returning an unsubscribe function.

**The gateway** (`main/gateway/`) does what web's `clientFetch` and mobile's `apiFetch` do, one process over:

| Piece | Does |
| --- | --- |
| `core` | Base URL (the real API, or the in-process mock dispatch in mock mode), JSON encoding and parsing, turns every failure into `ApiError` |
| `fetch` | Adds `Authorization: Bearer` from the session, the platform and version headers, refreshes and retries once on 401, maps network failures to the offline state |
| `realtime` | Opens SSE and WebSocket connections with the same headers, forwards messages to the subscribing window, closes them when the window closes or the session ends |

**`ApiError`** carries `status`, `code` (the backend's error code), `message` and, for validation failures, `fieldErrors` keyed by field path. It crosses the bridge as plain data and is rebuilt in `lib/api.ts`. Every failure is handled by status, the same way everywhere:

| Status | Handling |
| --- | --- |
| **401** | The gateway refreshes once, retries once; if the refresh fails, Main signs out (Section 5) |
| **403** | Show the no-access state |
| **404** | Show the not-found state |
| **422 / validation** | Map `fieldErrors` onto the form |
| **409, 429, and other client errors** | Show the backend's message in a toast |
| **5xx** | Toast with a retry |
| **Network failure** | Offline: cached data with the indicator, or the offline state; a mutation shows a retry toast |

Types have two eras, split at integration. **While a feature is mocked**, no backend exists to generate from, so its types are provisional: hand-written in the feature's `api/types.ts` from the shapes its screens imply, used by its hooks, fixtures and handlers. **At integration** they are deleted and the hooks import from `shared/api.ts`, generated from the backend's OpenAPI spec (`pnpm api:types`) and regenerated whenever the backend's api-docs change; from then on, nobody writes a type for a built endpoint by hand. **Unknown enum values are tolerated**: a `switch` on a backend enum always has a default branch that renders something sensible, so the backend can add a value without breaking an installed version — which, like a mobile build, may stay installed for months.

### Mock mode

Before a unit is integrated, its screens run on mock data that goes **through the real data layer**: the same hooks, the same bridge, the same gateway, the same types. Only where the requests go differs.

- `APP_ENV` other than `production` with `API_MODE=mock` makes the gateway's `core` dispatch each request to the handlers collected in `main/mocks/index.ts`, **inside the Main process**, with no server. With the mode off (`direct`, the default), `core` calls the configured API URL.
- **Passthrough.** A path with no handler is sent to the real API with the real `Authorization` header, so integrated units reach the real backend while mocked ones stay local. Integration removes a unit's handlers and its traffic falls through. The staging workflow's build step always sets `API_MODE=mock`: with passthrough, a staging build with no handlers left behaves as `direct`, and a later phase's units built before their backend exists run on their handlers without a workflow change. `main/mocks/index.ts` and the dispatch stay for the product's lifetime; production never carries them.
- **The unit that owns sign-in is integrated first.** Until it is, every environment runs on the mock auth and only its handlers answer, because the real backend would reject a mock token; after it, sign-in reaches the real backend and every passthrough call carries a real token. Nothing else is integrated before it.
- Each feature's `mocks/handlers.ts` maps `METHOD /path` to a resolver taking `{ params, query, body, session }` and returning the response, using the typed fixtures in `mocks/fixtures.ts`. The handlers are plain functions with no Electron imports, so Main runs them in mock mode and renderer tests run them behind the `window.api` double. `session` is the access token's decoded payload (`sub`, the user id); it is never verified, which is acceptable only because mock mode does not exist in production. Resolvers keep state in memory for the app's lifetime (a created record appears in the next list) and simulate latency.
- Fixtures are realistic: real-looking names, dates and amounts, every state the screen specs describe, edge cases (long text, empty lists, the maximum count). A mock that always succeeds hides the error states the design asked for, so handlers can be told to fail (`?mock=error`).
- **Mock auth** (`main/mocks/auth.ts`) handles sign-in, sign-out, refresh and `me` until the sign-in unit is integrated: it returns JWT-shaped tokens carrying the same claims the backend's tokens carry, through the same `session` module, so the gateway and the guards run unchanged. The sign-in unit's own `mocks/` replaces it when that unit is built, and that unit's integration removes it.
- **Real-time channels are not mocked.** In mock mode a screen that would subscribe polls its query against handler state instead; the real channel is wired at integration, as the api-docs specify.
- **Production builds carry no mock mode.** A `beforePack` hook referenced from `electron-builder.yml` fails a production build with the mode on, the gateway imports the mock module only outside production, and a test proves the mode is ignored when `APP_ENV` is `production`.
- **A unit not yet `Integrated` is unreachable in production.** Once the app is released, a later phase's built-but-not-integrated screens land on `main` before their endpoints exist. Each feature's `index.ts` exports its readiness; the router mounts a unit's routes and the shell shows its navigation entries only when `isIntegrated(area) || !isProduction`; `/desktop-integrate` flips the readiness when the unit is integrated; and a test proves no un-integrated route is reachable with `APP_ENV=production`.

## 4. Data layer

- **One hook per query, one per mutation**, in the feature's `api/`. Components never call `window.api.fetch` directly; they call hooks, and hooks call `lib/api.ts`.
- **Keys** are hierarchical and built by functions: `orderKeys.all`, `orderKeys.list(filters)`, `orderKeys.detail(id)`. Invalidate the narrowest key that covers the change.
- **Lists** use `useInfiniteQuery` for cursor pagination and `useQuery` with `placeholderData: keepPreviousData` for page numbers, matching whichever the api-docs give, rendered through `@tanstack/react-virtual` past ~100 rows. Filters and sort live in the route's search params when a screen should be reachable by link, otherwise in local state.
- **Mutations** invalidate on settle, not on success alone. Optimistic updates are for actions where the user expects instant feedback (toggles, reordering) and always roll back from the snapshot on error.
- **No retries on 401, 403, 404 or 422.** Retry other failures twice with backoff, queries only; mutations never retry on their own.
- **Derived data is computed with `select`** or a memo, never copied into state.
- **Real-time** updates write into the cache with `setQueryData` or invalidate, so every window reading that entity updates. Subscriptions open when a window that needs them is visible and close when it is not.
- **Persisted cache is off by default.** When the product needs data readable offline, the build plan turns on `persistQueryClient` with a Main-side encrypted store for the entities it names, excluding anything `security` calls sensitive, and sign-out clears it.

## 5. Session

The backend issues, rotates and checks tokens. Main carries them and triggers refreshes; the renderer only knows whether a session exists.

| Token | Lives in | Why |
| --- | --- | --- |
| Access (JWT, minutes) | Main's memory, in `main/session/` | Dies with the process; never crosses the bridge |
| Refresh (opaque, days) | `safeStorage`, encrypted by the OS | Survives restarts; unreadable without the user's OS login |

- **Launch:** the main window shows its loading state while `session` reads the refresh token, refreshes, and `useMe` resolves. A missing or rejected refresh token means signed out. A network failure is neither: the refresh token stays, the signed-in shell renders in its offline state (from the persisted cache where it is on), and the refresh runs when the connection returns. The loading state waits no longer than the startup budget.
- **Focus and wake:** returning from sleep or regaining focus refreshes if the access token is within a minute of expiry, reading `exp` without verifying it — the backend verifies.
- **The gateway** refreshes single-flight: when several requests fail together, they wait on one shared refresh. It skips the logic for the refresh route itself. If the refresh fails, it signs out.
- **Sign-in** opens the backend's sign-in page in the system browser with a PKCE challenge and the app's callback scheme; the deep-link module receives the one-time code, `session` exchanges it with the verifier for tokens, and the renderer is told the session exists. Email-and-password sign-in, where the product has it, posts through the gateway. **No sign-in form ever loads a third party's page inside the app.**
- **Sign-out** calls the backend, then clears in this order: the access token, the refresh token in `safeStorage`, the renderer's query cache and any persisted cache (through an event), and any store that holds the user's data. Windows return to the auth tree. Sign-in clears the query cache too, so the next user never sees the previous one's data.
- **Guards** are the router's: the `(app)` tree mounts only when `session.status === 'signedIn'`, the `(auth)` tree otherwise, from `useSession` fed by Main's events. **A guard is a convenience, never authorization**: the backend decides access on every request.
- **Current user:** one `useMe` query. Its roles show or hide UI through `RoleGate`; the backend enforces them.

## 6. Windows, routing and navigation

- **One main window** carries the app; secondary windows (a detached document, a settings window, a picker) are created by Main's `windows` module on request, load the same renderer at a specific hash route, and are listed in `docs/design/navigation.md`. Dialogs and sheets are in-window components, not OS windows.
- **Window state is Main's:** bounds, maximised and display are saved on move and resize and restored on launch, clamped to a visible display. A minimum size (`DESIGN.md` → Layout) is set on every window.
- **Title bars follow the platform:** `titleBarStyle: 'hiddenInset'` on macOS with the traffic lights left in place, `titleBarOverlay` on Windows, the native frame on Linux. The shell's own title area is a drag region (`app-region: drag`) with its controls marked `no-drag`. Fully custom frames are a `DESIGN.md` decision, never a default.
- **Routes** are hash routes from `createHashRouter`: the `(auth)` tree and the `(app)` tree, every screen with its loading, error and not-found states, and an error boundary per layout. Dynamic segments and search params are validated at the top of the screen with a Zod schema from the area's `schemas/`; an invalid one renders the not-found state, not a crash.
- **Deep links** (`{scheme}://…`) and files opened from the OS arrive in Main, which validates them, focuses or creates the right window, and tells it which route to navigate to. A link into a protected screen remembers its target through sign-in.
- **Multi-step flows** keep their step in the route, so a reload and the back action behave.
- **The application menu** is the native one, built by Main per platform (the app menu and `Cmd` shortcuts on macOS; File/Edit/View/Window/Help with `Ctrl` elsewhere), from one shortcut table in `shared/` that the renderer's `useShortcut` reads too, so a shortcut is defined once and shown correctly in menus and tooltips.
- **Close vs quit:** on macOS closing the last window leaves the app running (the dock icon and tray stay); on Windows and Linux it quits unless the product runs in the tray by design. Pending work (an upload, an unsaved form) asks before the window closes, through `beforeunload` in the renderer and `close` in Main.

## 7. Components

- **shadcn components are ours.** Edit them in `components/ui/` to match `DESIGN.md`; never wrap one to restyle it. Add components with the shadcn CLI, then adjust.
- **Variants with `cva`**, states with `data-*` attributes (`data-state="open"`), styling through Tailwind. No inline styles except for values computed at runtime.
- **One component per file**, named in PascalCase, file in kebab-case (`order-summary.tsx`). A screen is `{screen-name}-screen.tsx`; its parts live beside it in a folder once there are more than two.
- **Props are typed with an interface**, named `{Component}Props`, and destructured. Boolean props read as predicates (`isOpen`, `canEdit`). Callbacks start with `on`.
- **Composition over configuration**: slots and children rather than a dozen boolean props. A component that takes more than seven props is usually two components.
- **Every list has an empty state**, every async component a loading, error and offline state, as the design system specifies. A screen is not done with only its happy state.
- **Desktop users live on the keyboard.** Every action reachable by Tab, every list navigable by arrows, Enter activates, Escape dismisses, the shortcut table covers the primary actions and the command palette exposes the rest; focus is visible and returns to where it came from when a dialog closes. Context menus are native (`Menu.buildFromTemplate` through the bridge) so they look and behave like the platform's.
- **Accessibility is part of the component**: semantic elements, labels on every input, `aria-*` only where semantics do not already say it, contrast from the system, reduced motion respected. `eslint-plugin-jsx-a11y` is on and not disabled per line.
- **Platform differences are conventions, not features.** `process.platform` (exposed once as `window.api.platform`) decides shortcut labels, title-bar layout and dialog button order; both come from `DESIGN.md`. Every platform gets every screen and every state.

## 8. State

In this order; use the first that fits.

1. **Server state** — TanStack Query. Anything the backend owns.
2. **Route state** — hash route params and search params. The current entity, the active tab, a filter a link should reproduce.
3. **Form state** — React Hook Form. Never mirrored into a store.
4. **Local state** — `useState` in the component that owns it.
5. **Global UI state** — Zustand, one store per concern, for what survives navigation and nothing above fits: panel sizes, selection, an unsaved draft. Stores hold no server data and no derived data.
6. **Persisted preferences** — `electron-store` in Main, through `window.api.prefs`: theme, layout, recent files, window state. Never tokens, never server data; `security` says what may be persisted.

No React Context for state that changes often; it re-renders every consumer. Context is for dependency injection (theme, the query client).

## 9. Forms

- **One Zod schema per form**, in the feature's `schemas/`, derived from the unit's business rules and the screen spec's field constraints — same names, same limits. The type is `z.infer`, never written twice.
- **React Hook Form with `zodResolver`**, validating on blur and then on change, submit disabled while submitting, never while merely invalid. `Enter` submits from a single-line field; `Cmd/Ctrl+Enter` from a multi-line one.
- **Server validation wins.** A 422 maps `fieldErrors` onto the fields with `setError`; an error with no field goes to the form's root error.
- **Submission goes through a mutation hook.** The form never calls the bridge.
- **Unsaved changes** are guarded on navigation and on window close when the form is long enough that losing it hurts.
- Inputs are controlled through RHF's `Controller` only for components that need it (selects, date pickers); native inputs use `register`.

## 10. Effects

- `useEffect` is for synchronising with something outside React: a bridge event, a window event, a third-party widget. Not for fetching, not for deriving state, not for responding to a prop change that could be computed during render.
- **Every bridge subscription returns its unsubscribe** (`window.api.on` returns one); an effect that forgets fires its handler twice after the next render and leaks across windows.
- Every async effect guards against the component having unmounted.
- Event handlers do the work that an event causes. If logic runs "after X changes", ask whether it belongs in the handler that changed X.

## 11. Styling

- **Tokens, not values.** Colors, spacing, radii, shadows and type sizes come from the CSS variables `styles/globals.css` defines from `DESIGN.md`, exposed as Tailwind utilities. Arbitrary values (`w-[13px]`, `text-[#333]`) need a comment saying why the scale did not fit.
- **`cn()` merges classes**; conditional classes go through it, never through string concatenation.
- **Dark mode follows the OS** by default (`nativeTheme`, forwarded over the bridge); `globals.css` redefines the semantic variables under `.dark`, so a component writes `bg-background` once. A manual override, where the product offers one, is a persisted preference applied at the root, and Main's `nativeTheme.themeSource` follows it so native menus and dialogs match.
- **Windows resize from the minimum size to ultrawide.** Layouts flex; content keeps a maximum width (`DESIGN.md` → Layout) and centres beyond it; panels collapse below the breakpoints `DESIGN.md` names; nothing clips or becomes unreachable at 200% OS scaling or at the largest font setting.
- **Class order** is Prettier's (`prettier-plugin-tailwindcss`), so diffs stay readable.
- **Motion** follows `DESIGN.md`'s Motion section: durations and easings from its table, reduced motion respected everywhere (the renderer's `prefers-reduced-motion` media query, which Chromium maps from the OS setting; Main reads `systemPreferences.getAnimationSettings().prefersReducedMotion` only where a native animation needs it), layout animations only through Motion's `layout` prop.

## 12. Files, OS and the bridge

- **The renderer never sees a path it did not get from a dialog or a drop.** Main's `files` module opens dialogs, receives dropped paths, and reads or writes only under paths the user granted this session; it rejects traversal and symlinks out of a granted root. A file's content crosses the bridge as a buffer or a stream, never as a path to read elsewhere.
- **Large transfers do not cross the bridge in one piece.** Downloads go to disk through Main's `net`, uploads read from disk through Main; the renderer gets progress events and a result. A payload over ~1 MB on IPC is a design smell.
- **External links** go through `window.api.shell.openExternal`, which Main allows for `https:` (and `mailto:`) on a user action only.
- **Notifications** are the OS's, through Main; a click focuses the window and routes like a deep link. The tray shows the app's state and a short menu, never a second UI.
- **Clipboard** writes happen on an explicit user action; the app never reads the clipboard without one.
- **Updates:** Main checks on launch and every few hours, downloads in the background, shows the update banner the design system specifies, and installs on quit or when the user asks; the force-update screen (`useUpdateState` with `supported` / `soft` / `force` from the backend's minimum-version endpoint) blocks the app under `force` and opens the download page.

## 13. Performance

- **Startup is budgeted**: the main window shows within the budget `CLAUDE.md` records (default 1.5 seconds to a painted shell on a mid-range laptop, release build); Main defers everything not needed for the first window (tray, update check, menus' dynamic parts) until after `ready-to-show`.
- **The renderer stays responsive**: long lists virtualised, heavy computation in a Web Worker or in Main, no synchronous bridge calls (`invoke`, never `sendSync`).
- **Main stays responsive**: file and network work is asynchronous; CPU-heavy work (image processing, parsing large files) runs in a `utilityProcess` or a worker, never on Main's event loop, which drives every window and menu.
- **`memo`, `useMemo` and `useCallback` are applied where a profile shows the cost**, not by habit. React Compiler is on and makes most of them redundant.
- **Images** are sized for the display scale, cached by Chromium, served from the backend's CDN; local images are decoded in Main only when the renderer cannot.
- **Memory** is watched: a window that is closed releases its subscriptions and caches; a leak check (open and close the window ten times) is part of the setup verification.
- **The renderer bundle is checked** when a dependency is added and stays under the budget `CLAUDE.md` records (default 5 MB); the installer size is watched per release.

## 14. Errors and observability

- **Errors render, they do not vanish.** A failed query shows the error state with retry; a failed mutation shows a toast with the backend's message; an uncaught render error reaches the nearest error boundary, never a white window.
- **Sentry captures** in both processes: uncaught errors, boundary errors, failed mutations with the request's method and path (never its body), Main's unhandled rejections, and native crashes. Expected `ApiError`s (401, 403, 404, 422) and offline failures are not reported.
- **Logging goes through `electron-log`** in both processes, scrubbed as `security` says, rotated, at `warn` and above in production. No `console.log` in committed code.
- **User-facing messages come from the backend's message keys** where the business rules define them, and from `DESIGN.md` → Content and Voice otherwise.

## 15. Code style

- **TypeScript strict**, no `any`, no non-null assertions without a comment, no `as` casts to silence the compiler. Narrow with type guards; model variants as discriminated unions. Three `tsconfig` projects (main, preload, renderer) so a Node type can never leak into the renderer.
- **Imports** in groups, blank line between: Node and Electron (Main only), React, third-party, `@shared/`, `@/` aliases, relative, then styles. Type imports with `import type`. No default exports except where a tool requires them.
- **Naming:** `camelCase` for values and functions, `PascalCase` for components and types, `SCREAMING_CASE` for true constants, `use` prefix for hooks, `is`/`has`/`can` for booleans. Files `kebab-case`. Folders in `features/` match the slugs in `docs/feature-areas.md`. IPC channels `domain:action` (`files:open`, `session:signOut`).
- **Functions are small and do one thing**; a file over ~300 lines is a sign to split by responsibility.
- **Comments explain why, not what.** A `TODO` carries the ticket or issue it refers to.
- **Dead code is deleted**, not commented out. Git keeps the history.
- **Formatting and linting pass before commit**, enforced by a pre-commit hook; CI repeats the check.

## 16. Tests in this stack

What tests must prove is in `testing`. In this stack:

- Vitest with two projects: `renderer` (jsdom, `@testing-library/react`, `user-event`, `jest-dom`) and `main` (Node, with `electron` mocked once in `vitest.setup.main.ts`).
- Renderer tests render through `renderWithProviders` (a fresh `QueryClient` with retries off, the router's memory history, the theme) with **`window.api` replaced by the test double** in `src/renderer/src/test/api-double.ts`, which answers `fetch` from the shared mock handlers and lets a test override one handler or fail one call.
- Main tests exercise modules directly: the gateway against MSW (`msw/node`) with the fixtures; IPC handlers through the contract with valid and invalid payloads; `session` with a fake `safeStorage`.
- Test files sit beside the code (`order-summary.test.tsx`); E2E specs in `e2e/`, run with Playwright's `_electron.launch` against the packaged app.
