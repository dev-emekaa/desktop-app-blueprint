---
name: desktop-issue-fix
description: Resolve issues in the desktop app's built or integrated screens — bugs, crashes in either process, wrong behaviour, design corrections, accessibility and performance problems, platform-specific defects, and changes the backend's client handoff requires — through validation, a failing test first, a surgical fix, doc updates, verification on every platform, and a note on how the fix ships, since every fix is a new signed version through the update feed. Use for anything that changes existing UI, flows, OS wiring or integrations; not for designing, building or integrating a new unit.
---

# Desktop Issue Fix

## 0. Project Configuration

Read these from `CLAUDE.md`; ask once for anything missing and offer to record it.

| Key | What it is |
| --- | --- |
| **Project name** · **Audience · client** | Which product and client this repo is. |
| **Backend repo path** | For `pnpm docs:pull` and `pnpm docs:push`. |
| **Staging API URL** | For reproducing and verifying against the real backend. |
| **Dev server command** · **Test commands** · **E2E command** · **API types command** | How to run, test and regenerate. |
| **Platforms** | Which the product ships, so a fix is verified on each. |
| **Production status** | Not released yet / released. Decides how a visible change is rolled out, and whether a fix goes through a hotfix branch. |

## 1. Scope

This skill handles any change to what already exists:

- a bug or crash in a screen, flow or integration, found by me, by QA, by Sentry (renderer or Main), or by a user — on one platform or all;
- a change the backend's **client handoff** message requires, after `/backend-issue-fix` changed or added endpoints;
- a design correction (a screen drifted from its approved export or from `DESIGN.md`), an accessibility, performance or startup problem, a copy change, a signing, notarization or download-page correction;
- a small product change to an existing screen that I ask for directly, with the docs updated to match.

It does not design, build or integrate a new unit: that is `/desktop-build` and `/desktop-integrate`. A "fix" that needs a new screen is routed there. Preparing and rolling out a release is `/desktop-release`.

I will send one issue or a list. For each, follow Section 3.

## 2. Constraints

These apply to every change.

- **Standards as written:** `coding-standards`, `security`, `testing`, `DESIGN.md`; `ci-cd` where a workflow, `electron-builder.yml` or `electron.vite.config.ts` changes; `analytics` where installed. This skill does not restate them.
- **Diffs are minimal and surgical.** Touch only what the issue needs. No drive-by refactors, no restyling, no "while I'm here".
- **The docs stay true.** A fix that changes what a screen sends or shows updates its screen spec and hands it to the backend; a fix that changes how a screen looks updates its design export — edited on the unit's canvas when its tracker row records a link that opens, otherwise redesigned from the export and `DESIGN.md`, and exported again once I approve; a fix that changes an event updates `docs/analytics-events.md`; a fix that changes an environment variable updates `.env.example` and the infra guide; a fix that changes a bridge capability updates the contract and its tests; a fix that changes what the app stores or sends updates the privacy section of `docs/release/`. Docs never mention the issue; they describe the corrected behaviour as if it had always been so.
- **The backend's behaviour is not worked around.** If the issue is on the backend, it is logged in `docs/integration-handoff/backend-issues-desktop.md` (format in `/desktop-integrate` Step 5) and the client waits.
- **Every fix says how it ships.** There is no over-the-air layer: every fix is a new signed version through the update feed, which `/desktop-release` rolls out. A fix for the live release goes through a hotfix branch (`ci-cd` → Hotfixes) so unreleased work on `main` does not ship with it; a fix that can wait rides the next release from `main`. Say which before the fix is approved, and whether the fix touches Main, the preload or a native module, because those widen what must be verified and label the PR `e2e`.
- **Business rules are not reinterpreted in a fix.** A screen that enforces a limit differently from the rules is corrected to the rules; a rule that seems wrong is raised with me, not changed on the client.
- **No new dependency** outside `tech-stack` without asking. If a fix cannot be made without one, say which requirement it fails and wait.
- **Once the app is released**, a visible change to a flow users rely on ships behind the unit readiness switch, or a remote flag where the product has one, or in a coordinated release, as `ci-cd` → Releasing alongside the backend says, and never breaks a version that users still run: installed versions keep calling the backend for months.

## 3. Per-Issue Workflow

### Step 1 — Validate

1. **Pull first**: `pnpm docs:pull`, commit what it changed, so the spec and handoff file you will touch are the backend's current versions.
2. **Reproduce it**: in mock mode when the issue is client-side, against staging (`pnpm dev:staging`) when it involves the backend, on the platform it was reported on and then on the others. For a crash, start from the Sentry event: the stack, the breadcrumbs, the process it came from (renderer or Main), the OS and its version, the app version and the display scaling. A Main-process crash or a native-module crash is reproduced on the platform it came from before it is called fixed. Look at the screen; read the logs and the network calls. An issue that cannot be reproduced is reported as such with what was tried, not fixed speculatively.
3. **Confirm it against the docs**: the design export and `DESIGN.md` (is it a design drift?), the screen spec and app flow (is the screen doing what the flow says?), the business rules (is the limit or state right?), the api-docs (is the backend returning what it documents?).
4. **Classify it:**
   - **Client bug** — the code does not do what the docs say, on one platform or all. Fix here.
   - **Platform or OS defect** — the code is right and a platform, an OS version or a library misbehaves. Fix with the narrowest guard (`process.platform`, the OS version), record the version it affects, and note when the guard can be removed.
   - **Backend bug** — the endpoint does not do what the api-docs say. Log it; stop on this issue.
   - **Doc error** — the docs disagree with each other or with an approved design. Report it to me with the options; do not pick one.
   - **Change request** — the docs and code agree, and I want different behaviour. Treat as Step 2 with the doc updates included.
   - **Handoff adoption** — a backend handoff lists the change. Go to Section 4.

### Step 2 — Plan → **wait for my approval**

For anything beyond a one-line fix, list per issue: the root cause, the fix, the files, the tests to add or change, the docs to update, and anything the fix changes for the user. Collect the questions. Proceed only after my approval; trivial fixes (a typo, a token, a label) may be batched into one approval.

### Step 3 — Failing test first

Write the test that reproduces the issue at the lowest layer that can: a unit test for logic, an integration test for a screen state or an interaction, a Playwright journey only when the issue is in the packaged app, an OS capability or the real backend. A Main-process bug gets its test in the Main project, against the mocked `electron`. Run it; it fails.

### Step 4 — Fix

Make the change. Run the test; it passes. Run the suite for the unit and for anything that shares the changed component; everything passes. Look at the screen again on every platform the product ships, light and dark, at the minimum window size, in the state that was broken.

### Step 5 — Docs

Per the constraints: the screen spec (committed, then `pnpm docs:push --specs {file}` for the file that holds the changed screen — its canonical area file, or its phase file before promotion), the design export, `images.md`, `analytics-events.md`, `.env.example`, the infra guide, `e2e/README.md`, the privacy section in `docs/release/` — whichever the fix touched. A backend issue found here goes in the handoff file: write the entry, commit, `pnpm docs:push --handoff`; if the push is refused, pull (the file merges) and push again. `DESIGN.md` itself changes only through `/design-system`.

### Step 6 — Verify

Walk every flow the fix touches, in mock mode or against staging as the issue required, on every platform the product ships (on this machine's platform directly, on the others through the PR's `e2e` artifacts), and every flow that shares the changed component. **Present the walk in the chat**, screen by screen, and end with anything that failed. A failure is fixed through Steps 3 and 4 again.

### Step 7 — Report and deliver

For each issue: what it was, the root cause, what changed, the tests added, the docs updated, what the user will notice, and **how it ships** — the next release from `main`, or a hotfix branch from the live tag through `/desktop-release`. Then a PR: branch off `origin/main` (or the hotfix branch), push, give me the link. Never create the PR yourself. When I report the merge — in this session or the next — confirm that Pipeline 2's run on it passed, including its Playwright suite on every platform.

## 4. Adopting a Backend Handoff

A client handoff message from `/backend-issue-fix` lists items tagged `FIXED` / `NEW` / `UPDATED` / `REMOVED`, grouped by feature area, with an "Action required on the client side" section split into **Breaking** and **Deprecated**, and the list of docs updated.

1. `pnpm docs:pull`, commit what it changed, regenerate `src/shared/api.ts`, and read the handoff with the updated api-docs beside it.
2. **Breaking items first.** For each: the hook, handler, fixture, form schema or state wiring that reads the old shape, changed to the new one; a test proving the new shape; the screen looked at.
3. **Then `UPDATED` and `FIXED` items**: where a client workaround or a pending screen was waiting, finish the wiring and set the handoff file's entry to `verified`.
4. **`NEW` items** are integrated only where an existing screen in this unit already needs them per its spec; a new screen is `/desktop-build`'s work.
5. **`REMOVED` items**: delete the hooks and handlers, and anything that called them.
6. **Background behaviour** the handoff notes (state that now changes on its own) is reflected the way `/desktop-integrate` Step 3 says.
7. **Deprecations** the handoff lists are adopted so the backend can raise the minimum version later; the report names the first released version that no longer reads the old shape, so the floor for `X-Platform` is raised only to that version once its rollout has reached 100%.
8. Tests, verification against staging (Step 6), and the report, which names the backend commit verified.

## 5. Definition of Done

- [ ] Every issue reproduced and classified; backend issues logged, doc errors reported.
- [ ] Every client fix has a test that failed before and passes after.
- [ ] The suite passes: this unit's and everything sharing the changed components; the fix was looked at on every platform the product ships.
- [ ] Every doc the fix touched is updated and pushed, with no mention of the issue.
- [ ] The verification walk was presented and everything it caught is fixed.
- [ ] The report says how each fix ships; the PR link was given; Pipeline 2's run on the merge passed.
