# desktop-app-blueprint

**Design, build and release a product's desktop app for macOS, Windows and Linux with Claude Code, connected to the real backend.**

Every screen follows one design system. Each part of the app is designed for your approval, built and tested on realistic mock data, then connected to the backend's real endpoints as they're ready. Releases are signed and go out through an update feed, to a few users first and then everyone. Seven Claude Code skills and the rules behind them, for Electron.

**Works with** the [Backend Blueprint](https://github.com/dev-emekaa/backend-blueprint), which it is a client of: it reads the backend's product docs and API docs, and sends back screen specs and issues.

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
| `--analytics` | Also install the analytics rule, for products that track usage with PostHog |
| `--personal` | Install skills to `~/.claude/skills` instead of the project |
| `--force` | Overwrite existing files. Default is to skip them |
| `--dry-run` | Show what would change, write nothing |

There is one stack — Electron, React, TypeScript, electron-vite, Tailwind CSS, shadcn/ui, TanStack Query — so there is no `--stack` option. Re-running is safe: existing files are skipped and listed.

### What it writes

```
.claude/
  skills/        7 skills
  rules/         6 rules (plus analytics.md with --analytics)
CLAUDE.md        documentation map · audiences · project configuration
docs/
  product-docs/  business-rules/  app-flows/  api-docs/  api-testing/            arrive from the backend
  screen-specs/  design/screens/  integration-handoff/  infra/  release/         written here
```

`docs/feature-areas.md` isn't created — it arrives from the backend.

### After installing

1. Fill in the **audiences** and **project configuration** in `CLAUDE.md` — above all, the path to the backend repo and the app id. Anything you leave blank, the skill that needs it asks for once.
2. Run `/desktop-setup`. It builds the foundation and pulls the backend's docs.
3. Run `/design-system` once the PRD and feature areas are in.
4. Run `/desktop-build` for the first feature area.

Commit `.claude/` with your project: cloud sessions and scheduled runs only read project skills. `--personal` is for throwaway work.

### Updating

```bash
npx github:dev-emekaa/desktop-app-blueprint init --force
```

- Skills and rules are overwritten, including any edits you made — run it on a clean working tree and read the diff before committing.
- `CLAUDE.md` is never overwritten: it holds your project's values.
- To pin a version: `npx github:dev-emekaa/desktop-app-blueprint#v1.0.4 init`. If an update doesn't seem to land: `npm cache clean --force`.

---

## How the app gets built

Each step needs the one before it; a skill whose inputs don't exist yet stops and says so.

```
From the backend: feature areas · business rules · app flows · (later) API docs
 │
 ├─ /desktop-setup      the foundation, once; pulls the backend's docs
 ├─ /design-system      DESIGN.md, once
 └─ /desktop-build      per feature area, in order:
      first, once: the navigation, the app's shell, menus and shortcuts
      design   every screen it needs, designed for your approval        → Designed
      build    screens on mock data, on every platform, with tests      → Built
               → /screen-specs, sent to the backend                     → Specced
      ↓ once every area is Specced, the backend designs its database, then
        builds the areas in order, writing each one's API docs
    /desktop-integrate  real endpoints in place of the mocks            → Integrated
    /desktop-release    a signed version on the update feed, checked on real
                        machines, rolled out 10% → 100%

/desktop-issue-fix — any time, for anything already built
```

**Later phases** follow the backend's: the unit becomes a feature, and `/desktop-build` reads the existing screens before designing new ones.

### How the app is put together

The desktop app holds no business logic — the backend owns the data and the rules. The screens run in a locked-down window that can't touch the computer directly; a small native layer handles windows, menus, files, sign-in and updates, and makes every request to the backend. The screens never see the sign-in tokens, so a flaw in the interface can't reach your files or the session.

---

## The skills

### `/desktop-setup`
The foundation, once: the Electron project with a separate identity for development, staging and production; the locked-down window and the bridge between the screens and the native layer; sign-in through the system browser and the data layer; the mock API that lets screens run before the backend exists; menus, shortcuts, deep links and updates; tests, error tracking, the signed CI/CD pipelines; the docs sync with the backend repo; and a guide to the update feed, Apple, Azure and GitHub steps only you can do.
**Writes** the foundation, `docs/infra/infra-setup.md`

### `/design-system`
How everything looks, moves and reads in an app used for hours with a keyboard: researches the best desktop apps in the product's category, proposes a direction for your approval, then writes `DESIGN.md` — colours, type, every component and its states, the window on each platform, keyboard shortcuts, window sizes, motion, writing style and accessibility — and applies it to the code.
**Writes** `DESIGN.md`

### `/desktop-build`
Designs and builds one feature area's screens: lists every screen its flows need — including offline states and window sizes — designs them for your approval, then builds them for macOS, Windows and Linux on realistic mock data with images and tests, checks every flow on the packaged app against the designs, and writes the screen specs the backend builds from.
**Writes** the designs, screens, mocks, tests and screen specs

### `/desktop-integrate`
Replaces the mocks with the real backend once its API docs land: checks the screens against the specs, connects every endpoint, handles every documented error, logs anything the backend must fix, and walks every flow on staging on every platform.
**Writes** the integration, the backend-issues file

### `/desktop-issue-fix`
Changes what already exists: reproduces the issue on every platform, writes a failing test, fixes it, updates the docs it touched, says whether the fix waits for the next release or goes out as a hotfix, and adopts changes the backend hands over.
**Writes** the fix, its tests, the docs it changes

### `/desktop-release`
Takes a release from "what should ship" to "live for users": bumps the version, writes the release notes and the download page, runs the pipeline that signs and notarizes the installers, has you check them on real machines, and raises the rollout from 10% to everyone while the crash rate stays low on each platform. Covers the first release, and the optional Mac App Store and Microsoft Store builds.
**Writes** `docs/release/`

### `/screen-specs`
What each screen sends to the backend and shows from it — the same skill the Backend Blueprint ships, run here on the built screens.
**Writes** `docs/screen-specs/`

---

## The rules

Loaded by Claude Code at the start of every session — except `ci-cd.md`, which loads only when deploy files are touched.

| Rule | Covers |
| --- | --- |
| `documentation-conventions.md` | Which docs come from the backend, which this repo owns, and each screen's progress from Draft to Integrated |
| `tech-stack.md` | The tools — Electron, electron-vite, React, Tailwind, shadcn/ui, TanStack Query, electron-updater, electron-builder, Vitest, Playwright, Sentry — and the environments and update feeds |
| `coding-standards.md` | How the app is structured and written: the native layer and the bridge, data, sign-in, mock mode, windows, keyboard, offline, performance |
| `security.md` | The locked-down window, the bridge, tokens, data on the machine, deep links, updates, secrets, payments, personal data — and what the backend must provide |
| `testing.md` | The test layers, environments and data, the machines to check on, and a test for every bug |
| `ci-cd.md` | PR checks, signing, staging and production releases, rollback when installed versions can't be recalled, releasing alongside the backend |
| `analytics.md` *(with `--analytics`)* | Event naming, no personal data, consent |

---

## Working with the backend

- **Sharing docs:** `pnpm docs:pull` brings the backend's docs here; `pnpm docs:push` sends screen specs and backend issues there. `/desktop-setup` installs both, reading the backend repo's path from `CLAUDE.md`.
- **Issues:** `/desktop-integrate` logs what the backend must fix in `backend-issues-desktop.md`; the backend's `/backend-issue-fix` fixes it and hands back a message that `/desktop-issue-fix` adopts.
- **The backend's side:** it must meet `security.md` → Backend contract — tokens, sign-in through the app's own link, the minimum-version check, the download page and account deletion. Backend Blueprint v1.3.0 or later does.
- **Before the first release:** install the Backend Blueprint with `--store-client` and set its `Client release status`. From then on it stops making changes that would break versions already installed.
- **Another client for the same product:** each client repo is separate. `/design-system` takes the brand from the first client's `DESIGN.md`, so the product looks like itself everywhere.

---

## The folder version

A downloadable alternative to the installer, built from the same files:

```bash
npm run package
```

It writes `dist/desktop-app-blueprint/` and `dist/desktop-app-blueprint-v{version}.zip`. Its guide is `folder/README.md`.

### Releasing

```bash
npm version patch        # bumps package.json, commits and tags (patch · minor · major)
git push --follow-tags   # pushes the commit and the tag
```

The zip appears under the release's **Assets** as `desktop-app-blueprint-v{version}.zip` — that is the file to share, not the Source code archives. A tag that doesn't match `package.json` fails the build.

---

## Requires

- Node 22 or later, and Claude Code
- For releases: an Apple Developer Program membership, an Azure Trusted Signing account, and a Cloudflare R2 or S3 bucket for the update feed — `/desktop-setup`'s guide walks through them
- Optional, for design research: Mobbin, connected through its MCP. Without it, `/design-system` and `/desktop-build` research on the web, plus any screenshots you put in `docs/design/references/`
- For the `/design` canvas: a recent Claude Code signed in to a claude.ai account — without it, `/desktop-build` designs as HTML prototypes instead

**License** proprietary — all rights reserved.
