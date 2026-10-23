# desktop-app-blueprint

**Design-to-release desktop app workflow for Claude Code — the docs come first, the design system governs every screen, the backend's api-docs drive integration, and signed releases ship through an update feed.**

Seven Claude Code skills and the rules they depend on, installed into an Electron project as one pipeline: set up the foundation, write the design system, design and build each feature area's screens on mock data for macOS, Windows and Linux, spec them for the backend, integrate the real endpoints, fix what breaks, and take each release through signing, notarization, a machine check and a staged rollout. It is a frontend half of a pair: the [Backend Blueprint](https://github.com/dev-emekaa/backend-blueprint) writes the product docs this one reads, and reads the screen specs this one writes. The [Web App Blueprint](https://github.com/dev-emekaa/web-app-blueprint) and the [Mobile App Blueprint](https://github.com/dev-emekaa/mobile-app-blueprint) are its siblings for the other clients of the same product, and the [Ideas Blueprint](https://github.com/dev-emekaa/ideas-blueprint) produces the brief, PRD and logo all of them start from.

```bash
npx github:dev-emekaa/desktop-app-blueprint init
```

---

## Install

Run it in your project root. Nothing to install first — `npx` fetches this repo and runs it.

```bash
npx github:dev-emekaa/desktop-app-blueprint init [options]
```

| Option | Effect |
| --- | --- |
| `--analytics` | Also install the analytics rule. For products that track user behaviour with PostHog |
| `--personal` | Install skills to `~/.claude/skills` instead of the project |
| `--force` | Overwrite existing files. Default is to skip them |
| `--dry-run` | Print what would change, write nothing |

There is one stack — Electron, React, TypeScript, electron-vite, Tailwind CSS, shadcn/ui, TanStack Query — so no `--stack` flag. Add `--dry-run` to any command to preview it first. Re-running is safe: existing files are skipped and listed, never clobbered.

### What it writes

```
.claude/
  skills/                          7 skills
  rules/
    documentation-conventions.md   which docs arrive from the backend, which this repo owns, readiness
    tech-stack.md                  the tools, and the environments and feeds
    coding-standards.md            the three processes, the bridge and the gateway, mock mode, session, windows, how code is written
    security.md                    security policy for an installed app with OS privileges, and the contract the backend must meet
    testing.md                     test layers, what each proves, environments, the machine matrix
    ci-cd.md                       pipelines, signing, the update feed, tags, rollback — loads only when deploy files are touched
    analytics.md                   --analytics only
CLAUDE.md                          documentation map · audiences · project configuration
docs/
  product-docs/  business-rules/  app-flows/  api-docs/  api-testing/     arrive from the backend
  screen-specs/  design/screens/  integration-handoff/  infra/  release/   written here
```

`docs/feature-areas.md` is deliberately **not** created: it arrives from the backend with `pnpm docs:pull`, which `/desktop-setup` installs. `docs/screen-specs/` is created empty; `/screen-specs` fills it from the built screens.

### After installing

1. Fill in the **audiences** and **project configuration** blocks in `CLAUDE.md`. The backend repo path and the app id matter most. Anything you leave blank, the skill that needs it asks for once.
2. Run `/desktop-setup`. It builds the foundation and pulls the backend's docs.
3. Run `/design-system` once the PRD and feature areas are in.
4. Run `/desktop-build` for the first feature area.

### Skills are project-scoped by default

They install to `.claude/skills/` and are meant to be committed: cloud sessions, scheduled runs and Cowork read only project skills, and rules that travel without the skills that execute them are conventions with no tooling. `--personal` exists for throwaway work.

### Updating

In a project that already has it, the installer skips existing files by design, so an update needs `--force`:

```bash
npx github:dev-emekaa/desktop-app-blueprint init --force
```

`--force` overwrites, including local edits to a skill or a rule. Run it on a clean working tree and read the diff before committing. `CLAUDE.md` is never rewritten.

Pin a tag for a fixed version: `npx github:dev-emekaa/desktop-app-blueprint#v1.0.0 init`. If an update doesn't seem to have landed, `npm cache clean --force`.

---

## The pipeline

Each step is gated on the one before it. A skill whose inputs don't exist yet stops and says so.

```
Backend Blueprint writes: feature-areas · business-rules · app-flows · (later) api-docs
 │
 ├─ /desktop-setup    → the foundation, once; pulls the backend's docs
 ├─ /design-system    → DESIGN.md, once
 └─ /desktop-build    → per feature area, in order:
      Phase 0: navigation model, shell, menus and shortcuts, once
      Design:  screen inventory → references → canvas designs → your approval   [Designed]
      Build:   screens on mock data through the real data layer, every platform [Built]
               → /screen-specs → pushed to the backend                           [Specced]
           ↓   once every phase-1 area is Specced, the backend designs its schema,
               then builds the areas in order, writing api-docs per area
      /desktop-integrate → per area as its api-docs land: real endpoints,
                           orphan audit, backend-issues handoff                   [Integrated]
      /desktop-release   → a signed, notarized version on the update feed,
                           checked on real machines, rolled out 10% → 100%

/desktop-issue-fix — any time, for anything that already exists
```

**Later phases** follow the backend's: `/feature-requirements` and `/phase-review` there produce the FRD, the feature files and the frontend actions; here the unit becomes a **feature**, tracked in `docs/phase-{n}/ui-implementation-order.md`, and `/desktop-build` reads the existing screens before designing new ones.

---

## The architecture in one paragraph

The app is a **client**, like the web and mobile apps: the backend owns the data, the rules and the auth, and the desktop app holds no business logic. What makes it a desktop app is a thin **OS layer** in Electron's Main process — windows, menus, tray, files, secure storage, deep links, updates — and one thing more: Main is also the **network gateway**. Every request to the backend is made by Main; the renderer calls `window.api.fetch` over a typed, Zod-validated IPC bridge and never sees a token. The access token lives in Main's memory, the refresh token in `safeStorage`, and the renderer runs sandboxed with context isolation, no Node, no remote content and a CSP, so a flaw in the UI cannot reach the file system or the session. Before the backend exists, screens run on in-process mock handlers dispatched in Main through the same gateway; integration swaps the handlers for the real endpoints without touching the screens.

---

## The skills

### `/desktop-setup`
**The foundation, once.** The electron-vite scaffold with three TypeScript projects; `electron-builder.yml` and `electron.vite.config.ts` with a development, staging and production identity each; the typed IPC contract in `src/shared/ipc.ts` and the preload bridge built from it; the session module and the gateway in Main (bearer header, platform and version headers, single-flight refresh, offline as a state); TanStack Query wired to Main's online state and window focus; `electron-store` and `safeStorage`; generated API types; the in-process mock mode that lets screens run on realistic data before the backend exists, with passthrough to the real backend for everything already integrated; design tokens from `DESIGN.md`; windows with the hardened `BrowserWindow` options asserted by a test, menus and shortcuts, deep links with the single-instance lock, files, updates, tray and notifications where the product has them, the PKCE sign-in flow through the system browser; Vitest, MSW and Playwright on the packaged app; Sentry in both processes; the three CI/CD workflows with signing, notarization and the update feed; the docs sync with the backend repo; an infra guide for the feed, Apple, Azure and GitHub steps only you can do.
**Reads** the rules, `CLAUDE.md` · **Writes** the foundation, `docs/infra/infra-setup.md`

### `/design-system`
**The source of truth for how everything looks, moves, feels and reads.** Reads the product docs to understand what kind of app this is and how it is used — in a window, for hours, with a keyboard; researches competitors and the best desktop apps in its niche on Mobbin and against the macOS and Windows guidelines for the chrome, and reverse-engineers what makes them work; proposes a direction with a style tile for your approval; then writes `DESIGN.md` in the DESIGN.md format, linted with Google's checker — tokens that map onto shadcn's variables, every component with every state, the navigation model, the shell and title area per platform, keyboard and shortcuts, window sizes from the minimum to ultrawide, patterns for list, table, detail, form, onboarding, empty, error and offline, motion, imagery, content and voice, accessibility — and applies the tokens to the code. Takes the brand from a sibling client's `DESIGN.md` where one exists. Built to avoid the generic.
**Reads** PRD, brief, feature areas, rules, flows, brand assets · **Writes** `DESIGN.md`

### `/desktop-build`
**Design and build one unit's screens.** Derives every screen every flow needs, with its states — including offline, the permission-denied state and the window sizes — and gets the inventory approved; pulls references per screen type; designs the batch on the `/design` canvas (or as HTML prototypes where the canvas is unavailable) and iterates with you until approved; then builds the screens for macOS, Windows and Linux with realistic mock data through the real data layer, wires the OS capabilities each screen uses through the bridge, sources images (stock, looked at before use; generated; or a prompt in `images.md`), writes the tests and the Playwright journey, walks every flow on the packaged app against the approved designs, and runs `/screen-specs` so the backend can build. Two stages with a checkpoint at approval, or a whole phase's build stage in one session with subagents.
**Reads** flows, rules, DESIGN.md, existing screens · **Writes** designs, screens, mocks, tests, screen specs

### `/desktop-integrate`
**Replace mocks with the real backend.** Audits every screen against the specs and the api-docs first (build gaps, design gaps, missing endpoints, the backend-facing capabilities — the native OAuth flow, minimum version, deletion, the checkout return); aligns the hooks to the api-docs and removes the handlers; wires every documented error and background behaviour; audits for orphan endpoints; logs anything the backend must fix in a handoff file the backend reads; updates the tests to the real shapes; walks every flow against staging on every platform and names the backend commit verified. One unit per session, or a whole phase in order.
**Reads** api-docs, specs, api-testing, flows, rules · **Writes** the integration, the handoff file

### `/desktop-issue-fix`
**Change what exists.** Reproduces the issue on the platform it was reported on and then the others, classifies it (client bug, platform defect, backend bug to log, doc error to report, change request, backend handoff to adopt), writes the failing test — in the Main project for a Main-process bug — fixes surgically, updates every doc the fix touched, verifies the flows it affected on every platform, says whether the fix rides the next release or goes out as a hotfix from the live tag, and delivers a PR. Adopts a backend handoff breaking-items-first.
**Reads** the issue, the code, the docs · **Writes** the fix, tests, doc updates

### `/desktop-release`
**From "what should ship" to "live for users".** Decides the release — from `main` or a hotfix — on the tracker and the staging verification; bumps the version by PR; writes the release notes and the download page copy; runs the production pipeline, which signs, notarizes, smoke-tests and opens the feed at 10%; has you check the installers on the machine matrix — clean install and update, Gatekeeper, SmartScreen, the secret store, scaling — against the production backend; raises the rollout 10% → 25% → 50% → 100% while the crash-free rate holds per platform, halts to 0% when it does not; records every release. Covers the first release, with the download page written from scratch and the Windows reputation warning handled honestly, and the optional Mac App Store and Microsoft Store builds.
**Reads** the tracker, the code, `docs/release/`, Sentry, the feed · **Writes** `docs/release/`

### `/screen-specs`
**What each screen sends to the backend and shows from it.** The same skill the Backend Blueprint ships, unchanged, run here in its From designs mode on the built screens so the backend has the specs it builds against.
**Reads** the built screens, flows, rules · **Writes** `docs/screen-specs/`

---

## The rules

These install into `.claude/rules/`, loaded at the start of every session — except `ci-cd.md`, which loads only when a workflow, `electron-builder.yml`, `electron.vite.config.ts` or a Playwright file is touched.

- **`documentation-conventions.md`** — which docs arrive from the backend and are read-only here, which this repo owns, how screen specs change hands, the UI tracker and its six readiness values (Draft, Ready, Designed, Built, Specced, Integrated).
- **`tech-stack.md`** — the tools: Electron with Fuses and ASAR integrity, electron-vite, React with React Compiler, React Router in hash mode, Tailwind, shadcn/ui, TanStack Virtual, Motion, TanStack Query, fetch in Main's gateway over a typed IPC bridge, Zustand, electron-store, safeStorage, React Hook Form and Zod, electron-updater on an R2 or S3 feed, electron-builder with Developer ID, notarization and Azure Trusted Signing, Vitest and Playwright, Sentry, optional PostHog; Lottie and 3D only when a feature needs them. And the environments, identities and feeds.
- **`coding-standards.md`** — the architecture (the renderer renders, Main is the OS layer and the gateway, the backend decides), the three-process folder structure and dependency direction, the bridge contract and the gateway with offline as a state, mock mode, the data layer, the session and token lifecycle, windows, routing, deep links and shortcuts, components for the keyboard, state, forms, effects, styling from the minimum window to ultrawide with dark mode from the OS, files and the bridge, performance and startup, errors, code style, tests in this stack.
- **`security.md`** — the renderer as a browser (context isolation, sandbox, no remote content, CSP, Fuses), the bridge as the trust boundary, tokens and sessions, data on the machine, deep links and inbound data, transport, updates and supply chain, secrets, third-party SDKs, payments, logging, personal data, and the **backend contract**: tokens in the body for the platform header, the minimum-version endpoint and the download page, the native OAuth flow through the app's scheme, account deletion, status semantics, additive changes, real-time bearer.
- **`testing.md`** — static, unit (Main's modules against a mocked `electron`, the gateway against MSW), integration (the bulk, over a `window.api` double), E2E with Playwright on the packaged app on three platforms, smoke, manual QA on a machine matrix of Apple silicon, Intel, Windows 11 and Ubuntu; environments and data; the regression rule.
- **`ci-cd.md`** — PR checks with an unsigned packaging check and Playwright on demand, the automatic staging deploy that builds on every platform, signs on macOS and Windows and notarizes on macOS, publishes to the staging feed and moves `staging-current`, the manual production deploy of the verified commit, smoke-tested on the signed artifacts before any user receives them, opened at 10% and tagged `v*`, the hotfix branch, releasing alongside the backend, what rollback means when installed versions cannot be recalled.
- **`analytics.md`** *(with `--analytics`)* — `posthog-js` in the renderer with the CSP entry, explicit events named `object_action`, no personal data, identify and reset, consent, flags cached in electron-store.

---

## Working with the Backend Blueprint

The two repos share the docs through `pnpm docs:pull` and `pnpm docs:push`, which `/desktop-setup` installs and which read the backend repo path from `CLAUDE.md`. Pull brings the backend's docs here, including its copy of the screen specs once it owns them; `push --specs` sends one screen-spec file there and `push --handoff` sends this client's two backend-issues files (`backend-issues-desktop.md` and its `-resolved` archive — each client writes only its own). The backend's `/backend-issue-fix` reads its open entries itself, appends each resolution, and writes a client handoff message that `/desktop-issue-fix` adopts.

The backend must meet the contract in `security.md` → Backend contract: its `security.md` and services rule (v1.3.0 and later) state the token, native OAuth, deletion and download-page lines from its side, its build skill the minimum-version endpoint per platform; its api-docs document the platform and version headers and the error shapes. Install the Backend Blueprint with `--store-client`, so its deprecations register exists, and have its `Client release status` set before the first release: that is the line after which it stops making breaking changes.

Where the product also has a web or mobile client built with its blueprint, the client repos are independent: each has its own `DESIGN.md` (the later ones take the brand from the first), its own handoff file, and its own pipelines. The backend serves all of them.

---

## The folder version

The downloadable, manual alternative to the installer — the skills, rules and template laid out to copy into a project by hand, with its own README. Build it from this repo:

```bash
npm run package
```

It writes `dist/desktop-app-blueprint/` and `dist/desktop-app-blueprint-v{version}.zip`, from the same assets the installer uses, so the two never drift apart. Its guide lives in `folder/README.md`.

### Releasing

Every release is a version tag. Pushing one builds the zip and publishes it on the repo's **Releases** page:

```bash
npm version minor        # bumps package.json, commits, and tags v1.2.0 (patch · minor · major)
git push --follow-tags   # pushes the commit and the tag
```

The zip appears under the release's **Assets** as `desktop-app-blueprint-v1.2.0.zip` — that is the file to share. The **Source code** archives GitHub adds to every release are the repo itself, not the folder version. A tag that doesn't match the `package.json` version fails the build, so the zip's name always matches its contents.

---

**Requires** Node 22 or later, Claude Code, and for the releases an Apple Developer Program membership, an Azure Trusted Signing account and a Cloudflare R2 or S3 bucket for the update feed (the setup skill's infra guide walks through them). The `/design` canvas needs a recent Claude Code signed in to a claude.ai account; `/desktop-build` falls back to HTML prototypes without it. **License** proprietary — all rights reserved.
