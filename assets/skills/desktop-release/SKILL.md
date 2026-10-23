---
name: desktop-release
description: Take a release of the desktop app from "what should ship" to "live for users" — decide what the release is, bump the version, write the release notes and the download page copy, run the production pipeline, check the signed installers on real machines on every platform, open the rollout, raise it step by step while the crash-free rate holds, halt it when it does not, and record it. Use for the first release and every one after, hotfixes included; not for building features or fixing bugs.
---

# Desktop Release

## 0. Project Configuration

Read these from `CLAUDE.md`; ask once for anything missing and offer to record it.

| Key | What it is |
| --- | --- |
| **Project name** · **Audience · client** | Which product and client this repo is. |
| **App id** · **Platforms** | The app's identity, and which platforms the release must be checked on. |
| **Update feed** · **Download page** | Where the release is published and where new users get it. |
| **Signing** · **Release owners** | The Apple and Azure identities the pipeline signs with; who may run the production deploy. |
| **Store distribution** | None, or a store whose release this skill also prepares (Section 5). |
| **Production status** | Not released yet (the first release, Section 6) / released. |
| **Sign-in providers** · **Payments** · **Tray and background** · **File types** | Decide which flows the machine check must walk. |
| **Backend repo path** · **Production API URL** | To confirm the backend the release depends on is live. |
| **E2E command** | Playwright, for the smoke suite and the machine check's screenshots. |
| **Crash-free floor** | The rate below which the rollout halts; 99.5% unless `CLAUDE.md` says otherwise. |

## 1. Scope

`ci-cd`'s pipelines build, sign, notarize, publish and tag. A release needs more than a pipeline can do: a version worth releasing, honest release notes, a download page that matches the installers, a person's check on real machines, a rollout watched by someone who can halt it, and a record. This skill does those, in order, and stops for my approval at every point where something becomes public.

There is one kind of release. Every change — a feature, a fix, a dependency bump — reaches users as a new signed version through the update feed, ordered by its version number, and nothing reaches them any other way. The only choice is **where the commit comes from**:

- **A release from `main`** — the normal case: `staging-current`, verified on staging, carrying everything merged since the last release.
- **A hotfix** — a fix for the live version that cannot wait for the unreleased work on `main`: the `hotfix/vX.Y.Z` branch from `ci-cd` → Hotfixes, carrying one fix and a patch bump, verified by its own staging run, released by SHA. The manual checklist is reduced to the flows the fix touched.

Building features is `/desktop-build`'s work, fixing bugs `/desktop-issue-fix`'s. This skill changes no product code; a problem it finds goes to those skills first.

## 2. Reference — Governing Files

| File | What it is |
| --- | --- |
| The UI tracker and the staging deploy message | What is `Integrated` since the last release, and the commit on staging. |
| `docs/release/checklist.md` | The release checklist this skill runs, extended whenever a release teaches it something. |
| `docs/release/download-page.md` | The download page's copy: what the product is, the system requirements per platform, the installer per platform and architecture, how updates work, and the privacy section — what the app stores on the machine and what it sends — which `security` → Data on the machine and Personal data must be true to. Written on the first release, updated when the product changes. |
| `docs/release/releases/vX.Y.Z.md` | One record per release: the commit and backend commit, what changed, the machine check, the rollout timeline, incidents, and what the next release should learn. A hotfix is a release and gets its own record. |
| `docs/infra/infra-setup.md` | The feed, the signing identities and the GitHub environments; the steps only I can do. |
| Sentry, the feed, the download page | Crash-free rates per version and platform, the live `stagingPercentage`, the links new users see. |

**Standards**: `ci-cd` for the pipelines, tags and rollback; `security` → Updates and supply chain for what the feed and the signatures guarantee; `testing` → Machine matrix for where the check runs; `DESIGN.md` → Content and Voice for the release notes. Follow them as written.

## 3. Constraints

- **Nothing becomes public without my approval**: not the production deploy, not the rollout opening, not a rollout step. Running the production deploy is the first approval; opening the rollout past the pipeline's 10% and each raise are more.
- **A release is checked on real machines** against the production backend before its rollout is raised. A runner is not a machine: Gatekeeper, SmartScreen, the secret store and display scaling only show on one.
- **The backend goes first.** A release that needs backend changes waits until the production backend is live with them, and the record names that commit.
- **The download page matches the installers.** Every link resolves to the current version's artifact for its platform and architecture, the requirements are the ones the build enforces, and the privacy section describes what this version does.
- **Release notes are true and in the product's voice.** What the user will notice, from the tracker; never "bug fixes and improvements" when there is something to say. They name the backend commit the release was verified against.
- **Rollouts are gradual and watched.** No release goes to 100% without the crash-free rate holding at each step, per platform. A halt is not a rollback: `ci-cd` → Rollback says what can and cannot be undone — installed versions are never recalled, and a release is always fixed forward with a higher version.
- **All platforms release together** from one pipeline run. If one platform's build or check fails, the release does not open on the others: a user with two machines expects the same version on both.
- **The signatures are never worked around.** An installer that fails notarization, SmartScreen or signature verification is not shipped with instructions to bypass the warning; the identity or the build is fixed.

## 4. Workflow

### Step 1 — Decide the release → **wait for my approval**

1. `pnpm docs:pull`, commit what it changed. Read the tracker for everything `Integrated` since the last `v*` tag, and the staging deploy message for `staging-current` and its version.
2. Confirm the staging checklist passed on the machine matrix — including an update from the previous staging version — and the Playwright suite is green on `staging-current` on every platform. If not, stop: there is nothing to release.
3. Confirm every backend change the release depends on is live in production, and that every deprecation the backend's handoff listed is adopted in this commit.
4. Present: **from `main` or a hotfix**, and why; the changes, grouped by feature area, in user terms; the units built but not yet integrated, which stay unreachable in production and out of the release notes; the proposed version (semver: patch for fixes, minor for features, major for a restructure — every release needs a higher number, because `electron-updater` orders by version); the backend commit it depends on; anything in Section 3 that is not yet true.

### Step 2 — Prepare

1. **Version:** a PR that bumps `package.json` to the approved version — the pipeline never bumps it, so the release stays reproducible from its commit. For a hotfix the bump is on the hotfix branch. Give me the link; the release continues from the merged commit, which Pipeline 2 verifies on staging again.
2. **Release notes:** the "what's new" text, from the changes, in `DESIGN.md`'s voice, with the backend commit. It goes in the release record and, where the product shows notes in its update banner or on the download page, there too.
3. **Download page:** diff `docs/release/download-page.md` against the release — a changed minimum OS version, a new platform or architecture, a new file association, a change to what the app stores or sends — and update the copy. The links themselves come from the feed's `downloads.json` at 100%; the copy is updated here.
4. **Checklist:** run `docs/release/checklist.md` and present what passed and what did not.

### Step 3 — Build

Run the production deploy (`ci-cd` → Pipeline 3) with the commit's SHA, or ask me to if I am the release owner. It checks the version against the last `v*` tag, builds on every platform, signs on macOS and Windows, notarizes on macOS, uploads to the pending prefix, runs the smoke suite on every platform, and — only if every platform passes — promotes the artifacts to the production prefix with `stagingPercentage: 10` and tags `vX.Y.Z`. A failed smoke run stops here, nothing reached users, and the failure goes to `/desktop-issue-fix`.

From this moment one user in ten who checks for updates receives the release. That is deliberate: the pipeline's 10% is the first rollout step, and the machine check runs while it is live, with the halt one edit away.

### Step 4 — Machine check → **wait for my approval**

Only a person can install a signed build on a real machine, so this check is mine: you give me the checklist below per machine, I run it on the machine matrix from `testing` with the installers from the production prefix, against the production backend with the smoke account, and report what I saw; you record it and present it.

- **A clean install** on each: macOS opens the `.dmg` without a Gatekeeper warning (the notarization stapled); Windows runs the installer without a SmartScreen block (a "publisher unknown" warning on a young certificate is recorded, not ignored — Section 6); Ubuntu runs the AppImage and installs the `.deb`, and `safeStorage` finds the secret store.
- **An update** from the previous production version on each: the banner appears, the download completes, the restart installs, the session survives.
- The staging checklist's flows, read-only where they would create real data; sign-in with each provider through the system browser; the checkout return where payments are on; a deep link from another app; a file opened and saved through the dialogs; the tray and background behaviour where the product has them.
- The force-update screen, by pointing a machine at a test minimum version if the backend offers one.
- Light and dark following the OS; 100% and 200% display scaling; the minimum window size and a maximised window; a cold start within the startup budget; sleep and wake mid-session; the network cut mid-flow.

**Present the check in the chat** from my report, machine by machine, and end with anything that failed. A failure halts the rollout (Step 5.3), goes to `/desktop-issue-fix`, and the release starts again from Step 1 as a hotfix with the fix.

### Step 5 — Roll out and monitor

1. Watch Sentry's crash-free sessions and new issues for the new version, per platform, the backend's error rate for the new `X-App-Version`, and the update adoption the feed shows, at each step: after the first hour, the first day, and before each raise.
2. **Raise the rollout** — 10% → 25% → 50% → 100%, each step on my word, over days not hours — by editing `stagingPercentage` in the feed files with the feed script, only while crash-free sessions hold above the floor on every platform and no new issue affects a critical flow. At 100% the feed script writes `downloads.json` for the new installers; confirm the download page's links resolve.
3. **Halt** the moment a crash or a critical issue spikes on any platform: set `stagingPercentage` to 0 — users who have the version keep it — tell me, and route the fix through `/desktop-issue-fix` as a hotfix. A halted release is recorded with what happened, and it is never resumed: the fix ships as the next version.
4. At 100%: tell the backend the release is live with its version per platform, so it can plan the minimum-version change; update `CLAUDE.md`'s production status; close the release record with the timeline and what the next release should learn.

### Step 6 — Report

The version, from `main` or a hotfix; what shipped, by feature area; the machine check; the rollout timeline and the crash-free rate per platform at each step; incidents, if any; the backend commit the release runs against; and anything to carry into the next release. Commit `docs/release/` and deliver it as a PR: branch off `origin/main`, push, give me the link. Never create the PR yourself.

## 5. Store Distribution

Off by default, because a store changes the app's sandbox and takes updates out of its hands. Where `CLAUDE.md` turns it on, the store release is prepared **after** the feed release is at 100%, from the same commit:

- **Mac App Store:** the `mas` target with its own provisioning profile and entitlements (the App Sandbox, which `files.*` and `safeStorage` must already work under), `electron-updater` disabled in that build, a listing and screenshots in App Store Connect, the review account, and Apple's review. The record notes the review outcome.
- **Microsoft Store:** the `appx` target with the Partner Center identity, the listing, and the certification run.

Check each store's current submission requirements at the time, not from memory; they change. A store build is its own line in the release record, and a store rejection is classified the way `/desktop-issue-fix` classifies a report: a metadata problem is fixed in the listing, a bug goes to the skill, a policy question comes to me with the store's text.

## 6. The First Release

Before Step 1 can run the first time:

1. **The infrastructure exists** per the infra guide: the feed bucket and its domain, the Apple Developer ID certificate and notarization key, the Azure Trusted Signing account, the GitHub environments with their secrets, the release owners. Pipeline 2 has already proved the signing on staging; the production environment's identities are the same ones.
2. **Windows reputation:** SmartScreen judges a signing identity by how many installs it has seen, so the first releases may show a "publisher unknown" warning even with a valid signature. Say so early, record it on the download page honestly, and expect it to clear as installs accumulate; the warning is never bypassed with instructions to run anyway.
3. **The download page is written from scratch** in `docs/release/download-page.md` from the product brief and the design system's voice — requirements, installers, how updates work, the privacy section — and the page itself is served on the product's web domain, as the infra guide says, reading its links from the feed's `downloads.json`. Present it for approval.
4. **The checklist is written** from Sections 3 and 4 and from Apple's notarization and Microsoft's signing documentation, read at this moment, not from memory.
5. **The smoke account** is created on the production backend with the backend developer.
6. **The backend is told before the first release.** The Backend Blueprint is installed with `--store-client`, so its deprecations register exists, and the backend developer sets `Client release status` in the backend's `CLAUDE.md` to shipped: from then on every breaking change takes its deprecation path, and the minimum version for `X-Platform: macos`, `windows` and `linux` rises only to released versions.

Then Steps 1 to 6 as above. The first release opens at the pipeline's 10% and is held there longer, with the machine check on every machine in the matrix before the first raise.

## 7. Definition of Done

- [ ] The release was decided on the tracker and the staging verification, with the backend's production commit confirmed live.
- [ ] The version was bumped by a PR, the release notes and the download page copy match the release, and the checklist ran.
- [ ] The installers were checked on the machine matrix against the production backend — clean install and update, on every platform — and the check was presented.
- [ ] The rollout reached 100% on every platform with the crash-free floor holding, and the download page's links resolve; or it was halted and recorded.
- [ ] The backend was told the version is live; `CLAUDE.md` and the release record are updated; the PR link was given.
