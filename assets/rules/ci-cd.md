---
paths:
  - ".github/**"
  - "electron-builder.yml"
  - "electron.vite.config.*"
  - "e2e/**"
---

# CI/CD

How code moves from a pull request to users' machines. The checks and runners are named in `tech-stack`; what each test layer proves is in `testing`; the human steps around a release are `/desktop-release`'s.

## Principles

- **Every merge to `main` deploys to staging automatically.** Production deploys only by a manual trigger, after sign-off.
- **Production ships the exact commit verified on staging**, never "latest `main`".
- **Each environment is its own build**, because the API URL, the feed URL, the app id and the icon are baked in.
- **Every release is a signed binary** on macOS and Windows, and on Linux an artifact served only from the HTTPS feed. There is no over-the-air layer and no unsigned macOS or Windows artifact outside a developer's machine.
- **The app never deploys ahead of a backend change it depends on.** The backend deploys first and stays compatible; this client follows.
- **A release cannot be recalled**, so it is checked on real machines before it rolls out, rolled out gradually, and fixed forward.

## Environments

| | Development | Staging | Production |
| --- | --- | --- | --- |
| **Built by** | A developer: `electron-vite dev`, or `electron-builder --dir` for an unsigned packaged check | Pipeline 2, on every merge | Pipeline 3, manually |
| **Signed** | No | Yes — Developer ID and notarization, Azure Trusted Signing; Linux artifacts unsigned, HTTPS feed only | Yes, the same identities |
| **Distributed by** | Not distributed | The staging update feed; installers linked from the staging deploy message | The production update feed; installers on the download page |
| **Identity** | `{appId}.dev` | `{appId}.staging` | `{appId}` |
| **Version** | `package.json` | `{package.json version}-staging.{run number}`, a prerelease the staging app accepts | `package.json` version, which must be higher than the last `v*` tag |
| **Env values** | Local | GitHub `staging` environment | GitHub `production` environment, separate values |
| **Error tracking** | — | Sentry `staging` | Sentry `production` |

`electron-builder.yml` reads `APP_ENV` and sets the identity, name, icon, scheme, feed URL, Fuses and signing per environment; `electron.vite.config.ts` injects the same environment's constants. The feed lives on Cloudflare R2 or S3 under a per-environment prefix: the pipelines publish with electron-builder's `r2` publisher on R2 or its `s3` publisher on S3, and the app reads the `generic` URL. Builds run on GitHub's macOS (arm64 and x64), Windows (x64) and Linux (x64) runners, the Linux Playwright jobs under `xvfb-run` because Playwright's Electron launcher has no headless mode; the targets are `dmg` and `zip` on macOS, `nsis` on Windows, `AppImage` and `deb` on Linux.

Where the product distributes through the Mac App Store or the Microsoft Store as well, `/desktop-release` adds the `mas` and `appx` targets with their own identities and entitlements; neither is a default, because both change sandboxing and take updates out of the app's hands.

## Repo rules

- `main` is protected: no direct pushes, Pipeline 1 required to merge, one approving review where the team has more than one developer.
- Squash merge, so every commit on `main` is one reviewed PR.
- The version in `package.json` is bumped by the PR that prepares a release (`/desktop-release` Step 2), never by the pipeline, so every production build is reproducible from its commit.
- Staging deploys run one at a time, in merge order, so `staging-current` never moves backwards.
- Dependabot or Renovate PRs go through Pipeline 1 like any other; one that touches Electron or a native module is labelled `e2e`.
- The `staging` and `production` GitHub environments hold the signing identities, the notarization key and the feed credentials, and deploy only from `main` and `hotfix/*`; labelled-PR E2E runs build unsigned and need no secrets.

## Pipelines

| Pipeline | Trigger | What it does | Deploys to |
| --- | --- | --- | --- |
| **1. PR checks** | Pull request opened or updated against `main`; a push to `hotfix/*` | Static checks, unit and integration tests, an unsigned packaging check; Playwright on an `e2e` label | Nothing |
| **2. Staging deploy** | Push to `main` — every merge — except docs-only changes (`docs/**`, `*.md`); a push to `hotfix/*` | Re-runs the checks, builds artifacts on every platform (signed on macOS and Windows), publishes them to the staging feed, moves `staging-current`, runs the full Playwright suite | Staging |
| **3. Production deploy** | Manual — release owners only | Builds artifacts from the verified commit, signed on macOS and Windows, smoke-tests them, publishes to the production feed at a 10% rollout, tags the release | Production |

### 1. PR checks

**Required to merge.** Target: under ten minutes on Linux.

1. Install from the lockfile, cached, with `@electron/rebuild` for native modules.
2. In parallel: type check (three projects), lint and format check, unit and integration tests, dependency audit.
3. `electron-builder --dir` unsigned, to catch packaging configuration errors without signing.
4. On a PR labelled `e2e` (Electron or native-module changes, Main-process changes, navigation changes, anything risky): unsigned `--dir` builds on all three runners and the Playwright suite on each.

### 2. Staging deploy

Runs automatically, one at a time in merge order.

1. Re-run Pipeline 1's checks on the merged commit.
2. On each platform runner, build with `APP_ENV=staging`, the staging version and `API_MODE=mock` (passthrough sends every integrated unit's traffic to the staging backend), sign (and notarize on macOS), and upload the artifacts and feed files to the staging prefix.
3. Move the `staging-current` tag to this commit, only if every platform published.
4. Run the full Playwright suite on the staging artifacts, on every platform.
5. Upload the source maps to Sentry's `staging` environment.
6. Post the result to the team channel, if there is one: commit, version, pass or fail per platform, the installer links, and the changes since the last production release.

**After the workflow:** manual QA runs the staging checklist on the machine matrix, including an update from the previous staging version. A red Playwright run or a failed checklist means no production deploy until fixed. When the release relies on backend changes, tell the backend which backend commit was verified; that is the backend's production sign-off.

### 3. Production deploy

**Trigger:** GitHub → Actions → Deploy Production → Run workflow. Only release owners can run it; running it is the approval. Its one optional input, `commit`, defaults to `staging-current`; a commit SHA releases exactly that commit, and branch names are rejected, so `main` can never be deployed by mistake.

1. Check the actor is a release owner; stop if not. Show on the run page what is shipping: the commit, its message, the version from `package.json`, and the changes since the last release. Stop if the version is not higher than the last `v*` tag.
2. Check out the commit. Tests are not re-run: this commit already passed Pipelines 1 and 2.
3. On each platform runner, build with `APP_ENV=production`, sign and notarize, and upload the artifacts to a **pending** prefix the feed does not read.
4. Run the Playwright smoke suite on the pending artifacts, on every platform, read-only, with the smoke account.
5. If it fails, stop: nothing reached users; the pending artifacts are deleted.
6. If it passes: copy the artifacts to the production prefix and write the feed files with `stagingPercentage: 10`; upload the source maps to Sentry's `production` environment; add the `vX.Y.Z` tag. The rollout stays at 10% until `/desktop-release` Step 5 raises it while the crash-free rate holds; at 100% the feed script writes `downloads.json` next to the feed — the current version and the installer URL per platform and architecture — which the download page reads, so a release never needs a web deploy.

## Hotfixes

`main` usually holds unreleased work, so a fix for the live release cannot simply ride `main`: every unreleased merge would ship with it.

1. Branch `hotfix/vX.Y.Z` from the live release tag and cherry-pick the fix, with a patch version bump; the same fix is merged to `main` through a normal PR.
2. A push to `hotfix/*` runs Pipeline 2 against that branch: checks, a staging build on every platform, the Playwright suite, the manual checklist reduced to the flows the fix touched. It never moves `staging-current`.
3. Pipeline 3 is run with that commit's SHA.
4. The branch is deleted once the fix is live and on `main`.

## Releasing alongside the backend

- A release that needs new endpoints waits for the backend's production deploy, then ships. Order: backend, then the app, always.
- A breaking backend change is coordinated through the backend's deprecation machinery: the backend tolerates both shapes across the window, the app ships a version that reads the new one, and the backend raises the minimum supported version per platform only to that version, once it has rolled out to 100% and old versions have stopped calling. Installed versions run for months — users dismiss update prompts — so the window is never short.
- The release notes name the backend commit the release was verified against.

## Tags

| Tag | Moves? | Set by | Purpose |
| --- | --- | --- | --- |
| `staging-current` | Yes, on every successful staging deploy | Pipeline 2 | The commit live on staging |
| `vX.Y.Z` | Never | Pipeline 3 | A permanent record of each release, matching the version users see |

There is no staging branch; the tag is the record. Comparing two `v*` tags shows exactly what changed between releases. In a monorepo the tags carry the client: `staging-current-desktop`, `desktop-v…`.

**Version numbers** are semver: patch for fixes, minor for features, major for a restructure. Every release has one, because `electron-updater` orders releases by version.

## Rollback

| Situation | What can be done | How | Time |
| --- | --- | --- | --- |
| **Rollout in progress, problem found** | Stop it reaching more users | Set `stagingPercentage` to 0 in the feed files; users who have it keep it | Minutes |
| **Fully rolled out, problem found** | Nothing is recalled | Fix forward: a hotfix version through Pipeline 3; users update on their next check | Hours |
| **A version must not be run at all** | Force it out | The backend raises the minimum supported version above it; the force-update screen opens the download page | Hours, plus the time users take to update |

`electron-updater` does not downgrade (`allowDowngrade` stays off), so a previous version is never "republished": a release is always fixed forward with a higher version.

## Infrastructure

The R2 or S3 bucket and its public domain, the Apple Developer Program and its Developer ID certificate, the App Store Connect API key for notarization, the Azure Trusted Signing account, the GitHub environments and branch protection, and the download page are the steps only a person can do; `/desktop-setup` writes them into `docs/infra/infra-setup.md` so each environment can be set up start to finish from that doc.
