# Tech Stack — Electron · React · TypeScript

The technologies this desktop client uses. Use the current stable release of each; the lockfile pins versions, not this file. Anything not listed is added only after approval: the build plan names what the feature needs and recommends the most secure, efficient and standard option for this stack.

The app is the client only. The backend (NestJS or FastAPI, built with the Backend Blueprint) is the single authority for data, business rules and authentication. The app holds no business logic, is not the API, and runs no auth of its own. Its Main process is an **OS layer** — windows, menus, files, secure storage, updates, and the network gateway — never a second backend. macOS, Windows and Linux are one client with one codebase; the platforms differ in conventions, never in features.

## Application

| Need | Choice |
| --- | --- |
| Runtime | Electron, current stable, with Fuses hardened and ASAR integrity on (macOS and Windows) |
| Language | TypeScript in strict mode, in all three processes |
| Build · dev server | electron-vite: one config for `main`, `preload` and `renderer`, with HMR for the renderer and hot reload for Main |
| UI | React, with React Compiler on through electron-vite's React plugin |
| Routing | React Router (library mode) with `createHashRouter`, so the bundled `index.html` is the only document the app scheme serves and deep links have a route to land on |
| Styling | Tailwind CSS |
| UI components | shadcn/ui — copied into the repo and owned there, built on Radix primitives · `lucide-react` icons · `sonner` toasts |
| Long lists | `@tanstack/react-virtual` |
| Animation | Motion (`motion/react`): transitions, micro-interactions, layout and gestures |
| Animation, on demand | `@lottiefiles/dotlottie-react` for vector animations · React Three Fiber for 3D. Installed only when a feature needs them, never by default |
| Server data | TanStack Query in the renderer: caching, refetching, mutations, pagination, optimistic updates; online state from Main's `net.isOnline()` and `powerMonitor`, focus from window focus |
| HTTP | Native `fetch` **in the Main process**, through the project's `gateway` module; the renderer calls it over the typed bridge (`window.api.fetch`). Downloads and uploads with progress go through Main's `net` and `fs` |
| API types | `openapi-typescript`, generated from the backend's OpenAPI spec into `src/shared/api.ts`, imported by both processes |
| IPC | Electron `ipcMain.handle` / `ipcRenderer.invoke` behind `contextBridge`, from one typed contract in `src/shared/ipc.ts` with Zod payload schemas |
| Global UI state | Zustand, in the renderer — only for state the server does not own (panels, drafts, selection) |
| Persisted preferences | `electron-store` in Main: preferences, window state, recent files. Never tokens, never server data |
| Secrets on the device | Electron `safeStorage` (Keychain, DPAPI, Secret Service): the refresh token, and nothing else |
| Session | Access token in Main's memory, refresh token in `safeStorage`; the gateway refreshes. The renderer never holds a token |
| Forms · validation | React Hook Form · Zod, through `@hookform/resolvers` |
| Environment | Build-time constants per environment through electron-vite's `define` and `import.meta.env`, declared and validated with Zod in `src/shared/env.ts` |
| Mocking | This app's in-process mock handlers in Main for the build-before-integration phase (`coding-standards` → Mock mode) · the same handlers behind a `window.api` test double in renderer tests · MSW (`msw/node`) for the gateway's own tests |
| Dates | `date-fns` |
| Fonts | Bundled font files from `assets/fonts/`: Google Fonts downloaded as files, or other open-source fonts, as `DESIGN.md` names them. Nothing loaded from the network |
| Windows · menus · tray | Electron's `BrowserWindow`, `Menu`, `Tray`, `Notification`, `globalShortcut`, `dialog`, `shell`, `clipboard`, `powerMonitor` — through Main's capability modules |
| Window state | Persisted in `electron-store` by Main (bounds, maximised, display), restored on launch and clamped to a visible display |
| Deep links | A custom scheme registered by Main (`app.setAsDefaultProtocolClient`), with the single-instance lock, routed to hash routes |
| Sign-in providers | The backend's OAuth flow in the system browser with PKCE, returning to the app through the custom scheme · `safeStorage` for the resulting refresh token. No embedded sign-in |
| Payments | The backend's hosted checkout, opened in the system browser; the outcome comes from the backend. No embedded checkout window |
| Real-time | SSE and WebSocket opened by Main's gateway (`undici`'s fetch streaming, `ws`, or the Socket.IO client where the backend uses Socket.IO), forwarded to the renderer over IPC — as the api-docs specify |
| Updates | `electron-updater`, from a per-environment feed on Cloudflare R2 or S3 (the `generic` provider), with staged rollouts and differential updates |
| Packaging · signing | `electron-builder`: macOS (`dmg`, `zip`; Developer ID, hardened runtime, notarization), Windows (`nsis`; Azure Trusted Signing), Linux (`AppImage`, `deb`). `@electron/fuses` and ASAR integrity (macOS and Windows) at package time |
| Logging | `electron-log` in Main and renderer, scrubbed as `security` says, rotated; silent at `info` and below in production |
| Error tracking | `@sentry/electron`, Main and renderer, with native crash reports; source maps uploaded at package time |
| Product analytics *(optional)* | `posthog-js` in the renderer. The `analytics` rule applies when it is installed |
| Lint · format · types | ESLint (`typescript-eslint`, `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`, `eslint-plugin-testing-library`) · Prettier with `prettier-plugin-tailwindcss` · `tsc` per process |
| Package manager | pnpm, with a committed lockfile; its settings — the supply-chain ones `security` requires — in `pnpm-workspace.yaml`; `@electron/rebuild` for native modules |
| Unit and integration tests | Vitest (jsdom for the renderer, Node for Main) · React Testing Library · `@testing-library/user-event` · `@testing-library/jest-dom` |
| E2E and smoke tests | Playwright's Electron launcher, against the packaged app |
| Bundle analysis | `rollup-plugin-visualizer`, run by hand |

**Added when a feature needs them**, through the build plan: `@tanstack/react-table` for data tables, Recharts (shadcn charts) for charts, a rich-text editor, `better-sqlite3` with Drizzle for a local cache that works offline, `chokidar` for watching a folder, `node-pty` for a terminal, `sharp` in Main for image work, `electron-context-menu`.

## Services

| Need | Local | Staging | Production |
| --- | --- | --- | --- |
| The app | `electron-vite dev`: the renderer with HMR, Main reloading | Signed, notarized installers from Pipeline 2, on the staging update feed | Signed, notarized installers from Pipeline 3, on the production update feed |
| App identity | `{appId}.dev`, "{Name} Dev" | `{appId}.staging`, "{Name} Staging" | `{appId}`, "{Name}" |
| Update feed | — | `https://updates.{domain}/staging/` on R2 or S3 | `https://updates.{domain}/` on R2 or S3 |
| Backend API | Mock mode, passing un-mocked units through to staging | Staging backend; mock mode with passthrough while any unit is un-integrated | Production backend, direct |
| Error tracking | — | Sentry, `staging` environment | Sentry, `production` environment |
| Analytics *(optional)* | — | PostHog, staging project | PostHog, production project |
| CI/CD | — | GitHub Actions on macOS, Windows and Linux runners | Same |
| Download page | — | — | `{domain}/download`, served by the web client or the backend, reading `downloads.json` from the feed; linked from the force-update screen |

- **Three apps side by side.** Development, staging and production builds have different app ids, names, icons, schemes and update feeds, from one `electron-builder.yml` keyed by `APP_ENV`, so all three install on one machine and a tester never mistakes one for another.
- **Every release is a binary.** There is no over-the-air layer: a change ships as a new version through the update feed, differential where the previous version is known, installed on quit or when the user asks.
- **Environment values are baked in at build.** The API URL, the feed URL and the rest are build-time constants, so each environment is its own build.
- **Signing is not optional.** Unsigned macOS apps do not open and unsigned Windows apps are flagged by SmartScreen; `electron-updater` refuses unsigned updates. Every macOS and Windows artifact from Pipelines 2 and 3 is signed (Linux has no platform code signing; its artifacts are served only from the HTTPS feed); development builds are not.
- **Sentry** receives source maps at package time, scrubs what `security` says, and alerts on new issues per environment.
- **Media** is uploaded straight to the backend's bucket with presigned URLs the backend issues, and served from its CDN; the app never holds storage credentials.
