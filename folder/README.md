# Desktop App Blueprint

A docs-first workflow for designing, building, integrating and releasing desktop apps with Claude Code. Seven skills and the rules they follow take a product from its requirements to a designed, built, tested, integrated and signed Electron app for macOS, Windows and Linux — and every screen traces back to a reviewed document and an approved design.

It is a frontend half of a pair. The **Backend Blueprint** writes the product docs this one reads (feature areas, business rules, app flows, api-docs) and reads the screen specs this one writes. The stack is **Electron · React · TypeScript · electron-vite · Tailwind CSS · shadcn/ui · TanStack Query**, packaged and signed with electron-builder and shipped through an electron-updater feed.

---

## What's inside

```
desktop-app-blueprint/
  skills/                7 skills, one folder each
  rules/                 the rules every project uses, plus one optional
  templates/
    CLAUDE.md            the blocks to add to your project's CLAUDE.md
```

---

## Install

Everything goes into your project's `.claude/` folder, which Claude Code reads. In a terminal, from your project's root — set `BP` to wherever you unzipped this folder:

```bash
BP=~/Downloads/desktop-app-blueprint
mkdir -p .claude/rules docs
cp -R "$BP/skills" .claude/
cp "$BP"/rules/*.md .claude/rules/
```

Then:

1. **Remove the optional rule if you don't need it:** `analytics.md` — keep it only if the product tracks user behaviour with PostHog.
2. **Add the CLAUDE.md blocks.** Paste `templates/CLAUDE.md` into your project's `CLAUDE.md` (create the file if there is none). Fill in the audiences and the project configuration now — the backend repo path and the app id above all — or leave them; each skill asks for anything missing, once.
3. **Check it worked.** Start Claude Code and run `/context` — the rules appear under Memory files (`ci-cd.md` only once a deploy file is touched). Type `/` and the seven skills are listed.

> `.claude` is a hidden folder. To see it in Finder, press **Cmd + Shift + .**

**Commit `.claude/` with your project.** Cloud sessions and scheduled runs only read project skills, and the rules should travel with the code.

---

## Which skill when

**Starting a new desktop app**

1. `/desktop-setup` — the foundation, once: the electron-vite scaffold with three processes and a typed IPC bridge, app identity per environment, the session and the network gateway in Main, the data layer, mock mode, windows, menus, deep links and updates, tests, the signed build pipelines, and the docs sync that pulls the backend's docs into `docs/`.
2. `/design-system` — the design system, once, after the PRD and feature areas exist. Researches the niche's desktop apps, proposes a direction for your approval, writes `DESIGN.md` at the project root.
3. `/desktop-build` — one feature area per session, in the order the backend's feature areas give, or the build stage of a whole phase in one session once every area is designed. First its screens are designed on the canvas and approved; then they are built for macOS, Windows and Linux on mock data, tested, and specced for the backend.
4. The backend builds the area from your specs and writes its api-docs.
5. `/desktop-integrate` — one feature area per session, or the whole phase: replaces the mocks with the real endpoints and verifies against staging on every platform.
6. `/desktop-release` — the first release: the download page, the signed and notarized installers, the machine check, the staged rollout.

**Adding a new phase**

1. The backend's `/feature-requirements` and `/phase-review` produce the phase's FRD, feature files and frontend actions; `pnpm docs:pull` brings them here.
2. `/desktop-build` — one feature per session; it reads the existing screens before designing new ones.
3. `/desktop-integrate` — one feature per session, or the whole phase.
4. `/desktop-release` — a new version through the update feed.

**Fixing an issue, or adopting a backend change**

`/desktop-issue-fix` — reproduces on the platform it was reported on and then the others, classifies, writes the failing test, fixes surgically, updates the docs, verifies, says whether the fix rides the next release or goes out as a hotfix, and delivers a PR.

---

## The skills

**`/desktop-setup`** — Builds the foundation every screen stands on: the project per the tech stack, `electron-builder.yml` and `electron.vite.config.ts` with a development, staging and production identity each, the typed IPC contract and the preload bridge, the session module and the gateway in Main, TanStack Query wired to Main's online state, `electron-store` and `safeStorage`, generated API types, the in-process mock mode that serves screens before the backend exists, design tokens, windows with their hardened options, menus, shortcuts, deep links, files, updates, tray and notifications, sign-in through the system browser, Vitest and Playwright, Sentry, the three CI/CD workflows, the docs sync, and an infra guide for the feed, Apple, Azure and GitHub steps only you can do.

**`/design-system`** — Writes `DESIGN.md`, the source of truth for how every screen looks, moves, feels and reads on each platform: principles, colors and typography with contrast checked, spacing and the navigation model, the shell and title area, shape and depth, icons and imagery, motion, every component with every state, keyboard and shortcuts, window sizes, screen and state patterns including offline, content and voice, accessibility. Researches competitors and the best desktop apps in the niche first and proposes a direction for your approval.

**`/desktop-build`** — Designs and builds one feature area's screens: every screen every flow needs, with its states, inventoried for your approval; designed on the `/design` canvas and iterated with you; then built for macOS, Windows and Linux with realistic mock data through the real data layer, with the OS capabilities wired through the bridge, images sourced and checked, tests and a Playwright journey written, every flow walked on the packaged app against the approved designs, and the screen specs written for the backend.

**`/desktop-integrate`** — Replaces the mocks with the backend's real endpoints from the api-docs: an audit first, then the hooks aligned and handlers removed, every documented error and background behaviour wired, the backend-facing capabilities (the native OAuth flow, minimum version, deletion, the checkout return) integrated with the unit that owns them, an orphan-endpoint check, a handoff file for anything the backend must fix, tests on the real shapes, and a walk of every flow against staging on every platform.

**`/desktop-issue-fix`** — Resolves anything in what already exists: bugs, crashes in either process, platform defects, design drift, accessibility or performance problems, signing or download-page corrections, change requests, and the changes a backend handoff requires. Failing test first, surgical fix, docs updated, flows verified on every platform, and a clear answer on how the fix ships — the next release, or a hotfix from the live tag.

**`/desktop-release`** — Takes a release from "what should ship" to "live for users": the release decided on the tracker, the version bumped by PR, release notes and the download page copy, the production pipeline that signs, notarizes and smoke-tests, a check of the installers on real machines against the production backend, a rollout raised step by step while the crash-free rate holds and halted when it does not, and a record of every release. Covers the first release too, and the optional store builds.

**`/screen-specs`** — What each screen sends to the backend and shows from it, written from the built screens. The same skill the Backend Blueprint ships.

---

## The rules

Rules load into every Claude Code session and apply to all the code it writes.

| Rule | What it covers |
| --- | --- |
| `documentation-conventions.md` | Which docs arrive from the backend and which this repo owns, how screen specs change hands, the UI tracker, and the six readiness values from `Draft` to `Integrated`. |
| `tech-stack.md` | The tools the app uses and the environments, identities and feeds each build runs with. Anything not listed is proposed in a build plan and approved first. |
| `coding-standards.md` | The architecture, the three-process structure, the bridge contract and the gateway with offline as a state, mock mode, data layer, session and tokens, windows, routing, deep links and shortcuts, components, state, forms, styling across window sizes, files and the bridge, performance, errors and code style. |
| `security.md` | The renderer as a sandboxed browser, the bridge as the trust boundary, tokens and sessions, data on the machine, deep links, transport, updates and supply chain, secrets, third-party SDKs, payments, logging, personal data — and the contract the backend must meet. |
| `testing.md` | The test layers and what each proves, environments and data, the machine matrix, and the rule that every bug gets a test that reproduces it. |
| `ci-cd.md` | PR checks, the automatic staging deploy that builds on every platform, signs on macOS and Windows, and moves `staging-current`, the manual production deploy of the verified commit opened at a 10% rollout, the hotfix branch, and what rollback means when installed versions cannot be recalled. Loads only when deploy files are touched. |
| `analytics.md` *(optional)* | Explicit, named events with no personal data, identify and reset, consent, flags cached locally. |

---

**Requires** Claude Code, an Electron project (the setup skill creates it), and for the releases an Apple Developer Program membership, an Azure Trusted Signing account and a Cloudflare R2 or S3 bucket for the update feed. The `/design` canvas needs a recent Claude Code signed in to a claude.ai account; the build skill falls back to HTML prototypes without it. Updates replace these files — any edits you made to a skill or rule are overwritten, so keep your own changes in `CLAUDE.md` or new rule files.

© dev-emeka. All rights reserved. Licensed for use in your own projects; not for redistribution.
