## Documentation map

| Type | Location |
| --- | --- |
| Feature-areas registry *(from the backend)* | `docs/feature-areas.md` |
| PRD and product brief *(from the backend)* | `docs/product-docs/` |
| Business rules *(from the backend)* | `docs/business-rules/` |
| App flows *(from the backend)* | `docs/app-flows/` |
| API docs *(from the backend)* | `docs/api-docs/` |
| API testing *(from the backend)* | `docs/api-testing/` |
| Phase folders *(from the backend, plus this repo's UI tracker and the phase's screen specs)* | `docs/phase-{n}/` |
| UI tracker, first phase | `docs/ui-implementation-order.md` |
| Design system | `DESIGN.md` (project root) |
| Style tile *(only without the canvas)* | `docs/design/style-tile.html` |
| Navigation map | `docs/design/navigation.md` |
| Approved designs | `docs/design/screens/` |
| Image prompts | `docs/design/images.md` |
| Screen specs | `docs/screen-specs/` |
| Backend issues handoff | `docs/integration-handoff/` |
| Analytics events *(with the analytics rule)* | `docs/analytics-events.md` |
| Infra setup | `docs/infra/` |
| Release records | `docs/release/` |

## Audiences and clients

<!-- Copy the table from the backend's CLAUDE.md, then mark which audience this repo serves.
     An audience is a group of roles using its own separate product; a client is a separately designed app.
     macOS, Windows and Linux are one client, desktop. -->

| Audience | Primary | Clients |
| --- | --- | --- |

| This repo is | Value |
| --- | --- |
| Audience | |
| Client | desktop |

## Project configuration

<!-- The desktop skills read these. Anything missing, they ask for once; /desktop-setup fills in the commands and identifiers.
     Builds are signed in CI and shipped through an update feed. The backend repo path is used by the docs sync scripts. -->

| Key | Value |
| --- | --- |
| Project name | |
| Stack / framework | {{STACK}} |
| Backend repo path | |
| Sibling client repo path | the web, mobile or extension client's repo, where one exists — whichever client's `DESIGN.md` came first is the brand's source |
| Backend stack | |
| Backend OpenAPI spec | URL, once the backend is built — blank until then |
| Staging API URL | |
| Production API URL | |
| App id | `com.{company}.{product}` — `.dev` and `.staging` are derived |
| App scheme | `{product}` — `-dev` and `-staging` are derived |
| Platforms | macOS (arm64, x64) · Windows (x64) · Linux (x64) — which the product ships |
| Update feed | `https://updates.{domain}/` — the bucket and prefixes per environment |
| Download page | `{domain}/download`, and who serves it |
| Dev server command | |
| Test commands | |
| E2E command | |
| API types command | |
| Docs sync commands | |
| Design system | not written yet — `/design-system` updates this |
| Design canvas | /design (Claude Design in Claude Code) |
| Design reference library | Mobbin, through its MCP |
| Image generation | Gemini, through its MCP |
| Stock photo providers | Unsplash / Pexels — which are configured; their keys live in `.env` |
| Analytics | off |
| Sign-in providers | email and password · the backend's OAuth providers — which the PRD asks for |
| Payments | none · the backend's hosted checkout |
| Tray and background | none · tray icon · runs in the background — what the PRD asks for |
| File types | the file associations the app registers, if any |
| Store distribution | none · Mac App Store · Microsoft Store |
| Production status | not released yet |
| Production deploy trigger | Deploy Production workflow, run manually on the verified commit |
| Signing | Apple Developer ID team · Azure Trusted Signing account |
| Release owners | GitHub usernames allowed to run the production deploy |
| Startup budget | 1.5 seconds to a painted shell, mid-range laptop, release build |
| Bundle budget | 5 MB of renderer JS |
| Crash-free floor | 99.5% of sessions, below which a rollout halts |
