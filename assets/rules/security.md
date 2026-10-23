# Security

Security policy for this desktop client. The backend enforces every rule about who may do what; this app's job is to never leak, never trust what it did not verify, and never give the renderer more of the machine than it needs. A desktop app differs from a web page in two ways that shape everything here: it runs with the user's own OS privileges, so a flaw in the UI can reach the file system; and installed versions keep running for months after a release. The mechanism in this stack is in `coding-standards`; this file is the policy.

## The renderer is a browser, and is treated as one

Electron's own checklist, as hard rules. Every `BrowserWindow` is created with these, and a test asserts them:

- **`contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`, `webSecurity: true`.** The renderer has no Node, no `require`, no file system; it has `window.api` and nothing else.
- **Only this app's code loads in a window.** Production loads the bundled renderer; nothing remote is ever loaded in a window or a `<webview>`, and `<webview>` is disabled (`webviewTag: false`). A third party's page (sign-in, checkout, documentation) opens in the system browser.
- **Navigation is denied.** `will-navigate` and `setWindowOpenHandler` deny everything; `window.open` from the renderer never creates a window — Main does, from the contract.
- **A strict Content-Security-Policy** on the renderer: `default-src 'self'`, no `unsafe-eval`, no `unsafe-inline` for scripts, images and media from `'self'` and the backend's CDN, connections only to Sentry and analytics where installed (the API is reached by Main, so the renderer needs no `connect-src` for it). The packaged renderer is served over the app's own privileged scheme (`app://`, registered with `protocol.registerSchemesAsPrivileged` as `standard`, `secure` and `supportFetchAPI`, served by `protocol.handle`), never over `file://`: a response-header CSP does not apply to `file://` pages, and only a real scheme gives the renderer an origin. The policy is set as a response header on that scheme (and on the dev server), so it applies to every window, and a test asserts it is present in the packaged renderer.
- **Permission requests are denied by default** (`setPermissionRequestHandler`): camera, microphone, geolocation, notifications and the rest are granted only to features that use them, per window.
- **Devtools are closed in production**, and the keyboard shortcut that opens them is unbound.
- **Fuses** are set at package time: `RunAsNode` off, `EnableNodeCliInspectArguments` off, `EnableNodeOptionsEnvironmentVariable` off, `EnableCookieEncryption` on, `OnlyLoadAppFromAsar` on, `GrantFileProtocolExtraPrivileges` off (the renderer loads from `app://`, never `file://`), `EnableEmbeddedAsarIntegrityValidation` on (macOS and Windows; Linux has no ASAR integrity validation, so there the `OnlyLoadAppFromAsar` fuse and the HTTPS feed are the protection). A packaged app that fails these checks does not start.

## The bridge is the trust boundary

- **Every IPC handler validates its payload** with the contract's Zod schema before running, and checks the sender is one of this app's windows (`senderFrame` is non-null and its `origin` equals the app scheme's origin, or the dev server's origin in development — `app://` is why the packaged renderer has one). A renderer that is compromised can only call what the contract names, with what the schemas allow.
- **The bridge exposes capabilities, never primitives.** `window.api` has `files.open`, not `fs.readFile`; `shell.openExternal` with Main's allowlist, not `shell`; `session.signOut`, not the token. No handler takes a path, a URL or a command the renderer composed and acts on it unchecked.
- **Files are reached only through dialogs and drops.** Main reads and writes under paths the user granted in this session, rejects path traversal and symlinks out of a granted root, and never reads a path the renderer typed.
- **External links open only for `https:` and `mailto:`**, on a user action, after validation; `file:`, `javascript:` and custom schemes other than the app's own are never opened.
- **Events from Main to the renderer carry data, not instructions**: a deep link delivers a validated route, a notification click a route, an update event a state — the renderer decides what to render.

## Tokens and sessions

- **The access token lives in Main's memory only.** It is short-lived, used on every request, and dies with the process. It never crosses the bridge, never reaches a window, is never written to disk or a log.
- **The refresh token lives in `safeStorage` only**, encrypted by the OS keychain (Keychain, DPAPI, Secret Service) and bound to the user's OS account. If `safeStorage` is unavailable (a Linux session with no secret store), the app runs signed out per launch and says so, rather than writing the token in the clear.
- **Refresh is single-flight and bounded:** one refresh at a time, one retry per request, never for the refresh route itself. A failed refresh ends the session on this machine: tokens cleared, caches cleared, windows back to sign-in.
- **Sign-out is server-side first.** The backend revokes; Main then wipes what "Data on the machine" lists. Quitting the app signs nobody out; the refresh token is meant to outlive the process.
- **Sign-in runs in the system browser with PKCE**, never in a window of this app: the backend handles the provider, redirects to the app's scheme with a one-time code, and Main exchanges the code with its verifier. The `state` value is checked; a callback this launch did not start is dropped. Email-and-password sign-in, where the product has it, posts through the gateway.
- **Guards and role checks only hide screens and controls.** A screen that renders is not proof the user may see its data; the data came through the backend's checks. The cache is cleared on every sign-in and sign-out, so no role ever sees another session's data.

## Data on the machine

What may be kept on the machine, and where:

| Data | Where | Cleared on sign-out |
| --- | --- | --- |
| Refresh token | `safeStorage` | Yes |
| Preferences, window state, recent files | `electron-store` in `userData` | No, unless they identify the user |
| Drafts the user expects to find again | `electron-store`, in an entry encrypted with `safeStorage` | Yes |
| Server data for offline reading | The persisted query cache, encrypted with `safeStorage`, only for the entities a build plan names, never for the ones below | Yes |
| Files the user saved | Where the user chose, through a dialog | No |
| Logs | `userData/logs`, rotated, scrubbed | No |

- **Never persisted:** tokens, passwords, one-time codes, card data, government identifiers, account numbers, balances, private messages, documents the backend serves through signed URLs, and anything the screen spec marks as sensitive. These live only in the query cache, in memory.
- **`userData` is the app's folder and the only one it writes to on its own**; temporary files go to the OS temp directory and are removed when their purpose is served.
- **Clipboard and drag-out** of sensitive data happen only on an explicit user action; the app never reads the clipboard without one.
- **Sensitive screens hide behind the lock**: where the product handles money or health, a screen marked sensitive blanks when the OS session locks (`powerMonitor`'s `lock-screen`) and asks for re-authentication after the backend's inactivity window.

## Deep links and inbound data

- **The custom scheme is registered by Main** and the single-instance lock makes a second launch hand its arguments to the running instance; a link never starts a second copy of the app.
- **Every link argument is input.** Parsed with a schema before use; an unexpected shape is dropped and logged. A link names a route and ids; the screen fetches its data through the normal hooks. A sign-in callback is accepted only with the `state` this launch issued.
- **Files opened from the OS** (double-click, "Open with", drops) are treated like links: Main validates the path and type, grants it for the session, and tells the window; the renderer never receives a path it can act on.
- **Nothing is executed.** No `shell.openPath` on user-supplied paths, no `child_process` on user-supplied strings, no evaluation of downloaded code. An integration that needs a local tool runs a known binary with structured arguments and a validated allowlist.

## Transport and the gateway

- **HTTPS only.** The gateway refuses `http:` outside `localhost` in development, and the API, feed and CDN URLs are build-time constants, never configurable at runtime.
- **No certificate pinning by default.** It breaks on every certificate rotation and ships a reason to force-update; the OS trust store and the backend's TLS are the protection. A product that handles money or health may pin through the backend's published key set, as a build-plan decision with a rotation plan. `certificate-error` events are never overridden.
- **Every request carries only what it needs:** the bearer token, the content type, and the platform and version headers the backend's api-docs name (`X-Platform: macos`, `windows` or `linux`; `X-App-Version`, the app's version from `app.getVersion()`), so the backend knows which versions still call an old shape, per platform. Nothing user-specific in a URL.
- **Proxies are the OS's.** The app honours the system proxy and never ships its own proxy settings.

## Updates and supply chain

- **Every macOS and Windows artifact is signed**, and on macOS notarized, before it reaches any user; Linux artifacts are not code-signed by the platform and are served only from the HTTPS feed; `electron-updater` rejects an update whose signature does not match the running app's publisher, on macOS and Windows alike. A development build is unsigned and never leaves a developer's machine.
- **Signing identities live in CI, never in the repo:** the Developer ID certificate and the App Store Connect API key in the macOS job's environment, Azure Trusted Signing credentials in the Windows job's, each exposed only to the job that signs, never to pull requests.
- **The update feed is HTTPS on a domain this product controls**, per environment, and `electron-updater` is configured with that URL at build time. A staged rollout is how a release reaches users; a bad release is halted by setting its percentage to zero and fixed forward with a higher version.
- **The lockfile is committed and CI installs with `--frozen-lockfile`.**
- **New dependencies wait:** pnpm's `minimumReleaseAge` is set to seven days in `pnpm-workspace.yaml`, so a package version published an hour ago cannot be installed, which blocks most hijacked releases. A native module is reviewed with more care: it runs with the user's privileges, and it is rebuilt against this Electron with `@electron/rebuild` from source the lockfile pins.
- **Install scripts are off by default** (pnpm's `allowBuilds` map in `pnpm-workspace.yaml`, with `strictDepBuilds` on) and enabled per package with a reason — Electron itself and the native modules are the usual entries.
- **`pnpm audit` runs in CI**; a high or critical finding blocks the merge unless the PR records why it does not apply. Electron's own release is tracked: the app stays on a supported Electron major (the current and the two before it), because an older one stops receiving Chromium security fixes.
- **GitHub Actions are pinned to a commit SHA**, not a tag; Dependabot or Renovate opens update PRs that go through the same checks as any other.
- **Old versions keep running.** An installed version may call the backend for months, so a change that would break one is coordinated through the backend's deprecation and minimum-version machinery (`ci-cd` → Releasing alongside the backend), never shipped and hoped for.

## Secrets and environment

- **The app holds no secrets.** No keys to the backend's providers, no storage credentials, no signing keys, no webhook secrets. Anything that needs a secret is a backend endpoint. The values compiled in — the API URL, the feed URL, the Sentry DSN, the analytics key — are public by design.
- **Secrets never enter the repository**: not in `.env` files that are committed, not in `electron-builder.yml`, not in test fixtures, not in CI logs. Build-time values live in GitHub environments per deployment target.
- **Source maps** are uploaded to Sentry at package time and not shipped in the installer.

## Third-party SDKs

- **Every dependency that runs native code or reaches the network is a decision**, listed in `tech-stack` or a build plan with what it is for and what it collects, because it runs inside the user's session with the user's privileges.
- **Analytics carries no personal data:** no emails, names or free text in events or properties; the user is identified by the backend's id only after sign-in and reset on sign-out; no device fingerprinting beyond the OS and app version. Consent is asked where the product's users require it, and nothing is sent before it. The `analytics` rule has the detail.
- **Sentry scrubs before sending:** authorization headers, tokens, request and response bodies, form values, file paths outside the app's own, usernames in paths, and every field the privacy section names. Native crash reports are sent without the process memory.

## Payments

- **Payments happen in the backend's hosted checkout in the system browser**, never inside the app; the outcome comes from the backend's state (polled or pushed), never from a URL parameter in the return link.
- **Amounts shown come from the backend**; the app never computes a price the user will pay. It formats.

## Errors and logging

- **Users see a message, never a stack trace, a path or an internal id.** The error boundary renders the designed error state; the detail goes to Sentry.
- **Logs are scrubbed and bounded**: no request bodies, no user data, no tokens, no full file paths outside `userData`; rotated at a fixed size; `warn` and above in production.
- **Error states fail closed**: on a 401 or 403 a screen shows nothing from the cache for that entity. Being offline is not a verification failure; the offline state in `coding-standards` applies.

## Personal data

- **Collect only what the screen spec asks for**, and show only what the user's role needs for the action in front of them.
- **No personal data in links, analytics, logs, Sentry or error messages.**
- **Account deletion is in the app**: the flow renders what the backend provides, with the backend's re-authentication step, and sign-out and a local wipe follow. **Data export** renders the backend's signed link and saves through a dialog.
- **The uninstaller leaves nothing sensitive behind**: `userData` holds nothing a user would mind leaving, because tokens and sensitive data were never written in the clear.

## Backend contract

What this client assumes the backend provides. Items 1, 3 and 4 are lines in the Backend Blueprint's security and services rules (v1.3.0 and later); 2 and 6 are in its build skill; 5 is what its api-docs document per endpoint; 7 is its security rule. A backend that differs is a conflict to raise before the first integration, never a workaround on this side.

1. **Tokens in the body:** for a request carrying the platform header (`X-Platform: macos`, `windows` or `linux`), sign-in and refresh return the access and refresh tokens in the JSON response, not as cookies; sign-out revokes the session. The access token is a short-lived JWT, the refresh token is opaque and rotated on every refresh, with the grace window of about ten seconds for the token just rotated.
2. **Minimum supported version:** an endpoint callable without a session, which the app calls on launch with its platform and version, returning `supported`, `soft` or `force`, held per platform. The download page the force-update screen opens is this product's, in `CLAUDE.md`.
3. **Native sign-in:** an OAuth flow for native clients — the app opens the backend's sign-in URL with a PKCE challenge and its registered scheme; the backend handles the provider and redirects to that scheme with a one-time code; the app exchanges the code and verifier for its tokens.
4. **Account deletion:** an endpoint, behind re-authentication, that the app's deletion flow calls.
5. **Errors as documented:** 401 means no valid session (refresh or sign in), 403 means a valid session without permission (never refresh), and validation failures and every client-actionable error carry the shape and stable code the api-docs document for that endpoint.
6. **Additive changes only** once this client is released, or a coordinated deploy with a deprecation window for a breaking one, as the backend's build rules say; the backend reads the platform and version headers to know which versions still call an old shape, and raises its minimum only to a released version.
7. **Real-time connections accept the bearer token** in the handshake or a header, as they do for any native client.
