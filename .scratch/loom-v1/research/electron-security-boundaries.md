# Electron security boundaries for local agent execution

**Question:** What current Electron security guidance and supported APIs should shape Loom's main/renderer separation, preload bridge, secure credential storage, filesystem/process access, CSP, and update posture when it launches Pi and Git for a user-selected Project?

**Scope and method:** researched 7 August 2026 against Electron's official documentation only. The recommendations below are Loom-specific design conclusions drawn from that guidance; they are not claims that Electron alone restricts Pi or Git.

## Decision-ready conclusion

Loom should treat its React renderer as untrusted presentation code and make the Electron main process the only local privileged boundary. The main process owns the selected project root, API-key encryption, Pi/Git process lifecycle, and all filesystem access. A small, typed preload API should expose only named user-intent operations and read-only session/event data; it must never expose raw Electron, Node, IPC, filesystem, shell, process-launching, or arbitrary-path/command capabilities. The main process must then independently authorize every request against the persisted project connection and active ticket/worktree.

This does **not** sandbox Pi or Git. Those programs necessarily run with the desktop user's operating-system permissions. Loom's product safety policy must therefore constrain executable, arguments, `cwd`, environment, worktree, cancellation and output handling before spawning them. Electron's sandbox protects the renderer; it is not a substitute for that command policy.

## Recommended v1 boundary

| Boundary | Loom decision |
| --- | --- |
| Renderer | React UI only. `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, `webSecurity: true`; load packaged UI via a restrictive custom `loom://` protocol, not `file://`. Treat UI/markdown/log output as untrusted data and never execute it. |
| Preload | A tiny, versioned `window.loom` capability interface implemented with `contextBridge`: e.g. select/connect a project, request a scoped action, pause/resume/cancel a known session, subscribe/unsubscribe to sanitised events, and query already-authorised view data. It performs shape checks, but makes no privileged policy decisions. |
| Main | Sole owner of `safeStorage`, file dialogs, project-root canonicalisation, Git/Pi launch and termination, backend tokens, and IPC handlers. Validate each handler's sender frame and all IDs/paths; resolve every candidate path and ensure it remains within the selected project/worktree before I/O or spawn. |
| Agent subprocesses | Spawn only a fixed Pi executable and Git executable with argv arrays (never a renderer-supplied shell string), a controlled environment, and `cwd` equal to the ticket's allocated worktree. Do not pass provider keys to the renderer or backend. Capture/stream redacted output through explicit events. Kill the known child/process group on pause/cancel; record PID/session state for recovery. |
| Remote/auth UI | Prefer the packaged app for Loom UI. If an external identity or provider page is ever displayed in-app, put it in a separately configured remote view with Node integration disabled, isolation and sandboxing enabled, explicit HTTPS origin allowlists, denied-by-default permission handlers, no `webview` unless unavoidable, and no project/credential bridge. |

Electron says renderer sandboxing is enabled by default from Electron 20, but recommends enabling it in all renderers; Node integration disables it. It also says that disabling context isolation disables sandboxing. Set the values explicitly and test the packaged build rather than relying on defaults. [Process Sandboxing](https://www.electronjs.org/docs/latest/tutorial/sandbox) · [Security checklist](https://www.electronjs.org/docs/latest/tutorial/security)

## Main, renderer, preload, and IPC

- Electron warns that arbitrary remote content combined with Node integration is a severe risk. Only packaged local content should execute privileged code; any remote view must have `nodeIntegration` disabled and context isolation enabled. [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Context isolation and `contextBridge` are the supported separation mechanism. Electron specifically warns against exposing `ipcRenderer.send` or raw event callbacks: expose one narrowly scoped function per IPC operation and remove the IPC event object before forwarding a subscription callback. Bridge values are copied/frozen (functions are proxied), which reinforces a message/capability API rather than shared mutable state. [Context Isolation](https://www.electronjs.org/docs/latest/tutorial/context-isolation) · [contextBridge](https://www.electronjs.org/docs/latest/api/context-bridge) · [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Main-process IPC handlers must validate the sender. Electron notes that any web frame can potentially send IPC and recommends URL-parser plus allowlist validation. For Loom's local protocol, validate the exact expected `loom://` frame/origin and reject frames from dialogs, remote content, devtools, or unexpected windows. Validate action/session/project IDs against the authenticated local app state as well; sender validation alone is not authorization. [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Register a `will-navigate` allowlist and `setWindowOpenHandler` that denies by default. When a user explicitly opens an external help/auth link, validate an HTTPS allowlist before `shell.openExternal`; Electron says untrusted input to that API can compromise the host. [Security](https://www.electronjs.org/docs/latest/tutorial/security)

## Filesystem and process access

Electron's guidance is to keep privileged access out of web contents. Its renderer sandbox limits most system-resource access, whereas the main process remains privileged; therefore project selection, canonical-root checks, worktree creation, file access, spawning, and credential decryption belong only in main. This is an architectural inference from Electron's model, not a claim that Electron will confine a child process to a directory. [Process Sandboxing](https://www.electronjs.org/docs/latest/tutorial/sandbox) · [Security](https://www.electronjs.org/docs/latest/tutorial/security)

The supported Electron `utilityProcess` API is a main-process API that can create a Node-enabled child with controlled `args`, `cwd`, `env`, and stdio. It is suitable only for a Loom-controlled Node helper—not an automatic sandbox for an arbitrary external Pi/Git executable. If used, leave `allowLoadingUnsignedLibraries` disabled; Electron says to enable it only when specifically needed. The implementation ticket should choose the appropriate Node spawning primitive for Pi/Git, but preserve the boundary above. [utilityProcess](https://www.electronjs.org/docs/latest/api/utility-process)

Concrete enforcement requirements for the command-policy decision:

- Renderer inputs identify a server/local record (`projectId`, `ticketId`, requested lifecycle action), not an executable path, shell expression, working directory, arbitrary environment variable, or output-file path.
- Main resolves the ticket's worktree from its own records, rejects a missing/non-Git/out-of-root target, and launches the fixed executables using argument arrays with a minimal allowlisted environment.
- Limit concurrent sessions, bytes retained/forwarded, execution duration, and restart behavior; redact tokens from logs and never turn agent output into an instruction to the main process.
- User-confirmed operations remain the only route to external links, new project roots, provider-key entry/replacement, and any broadened filesystem/process capability.

## Credential storage

Use Electron main-process `safeStorage` to encrypt the device-local provider credential before persisting its ciphertext in app data; use the asynchronous encryption/decryption APIs. Electron recommends those APIs because they are non-blocking and support key rotation and temporary unavailability. The renderer receives only status such as “configured”, never plaintext; the backend never receives the provider key. [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)

At setup and startup, check encryption availability. On Linux, Electron documents that an unavailable secret store can fall back to an unprotected `basic_text` backend; `getSelectedStorageBackend()` exposes that condition. Loom should refuse persistence (or require the user to configure a supported OS secret store) rather than silently store a provider key with that fallback. Windows DPAPI protects against other machine users but not necessarily other applications in the same userspace, so this is protection of data at rest, not a reason to expose the key to renderer code. [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)

## Content, permissions, and CSP

- Serve the packaged React application through a custom privileged-but-restricted `loom://` protocol. Electron recommends custom protocols rather than `file://`, because `file://` pages can access arbitrary local files and XSS can exploit that. Register the scheme before `app.ready`, map only a known packaged asset root, reject traversal, and give it only the privileges Loom needs. [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Apply a production CSP to the packaged UI. Start from `default-src 'self'; base-uri 'none'; object-src 'none'; frame-ancestors 'none'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'self' https://<loom-api-origin>` and narrow it after the actual API/origins are chosen. Electron recommends a restrictive CSP, e.g. `script-src 'self'`; a custom protocol also enables a header-based policy instead of relying on a `file://` meta-tag exception. [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Keep `webSecurity` enabled; do not enable insecure-content, experimental, or Blink feature switches. Electron notes that disabling web security also disables same-origin protections and enables insecure content. [Security](https://www.electronjs.org/docs/latest/tutorial/security)
- Install `session.setPermissionRequestHandler` (and permission-check handling where applicable) for every remote-capable session, with deny-by-default behavior and an explicit origin/permission allowlist. Electron states that without a custom handler, permission requests are automatically approved. The packaged Loom UI should need none. [Security](https://www.electronjs.org/docs/latest/tutorial/security) · [session](https://www.electronjs.org/docs/latest/api/session)

## Packaging and update posture

The desktop installer is part of Loom's local privilege boundary. Ship current Electron releases and maintain a dependency update/security-response cadence; Electron says new framework versions include Chromium/Node/Electron security fixes. [Security](https://www.electronjs.org/docs/latest/tutorial/security)

For release builds, code-sign Windows and macOS artifacts; Electron recommends signing distributed apps, and macOS distribution also requires notarization. Do not present unsigned developer builds as production installations. [Code Signing](https://www.electronjs.org/docs/latest/tutorial/code-signing)

If v1 includes automatic updates, use an HTTPS-controlled update feed, run updater code only when `app.isPackaged`, handle updater errors, and notify/obtain a user restart to apply a downloaded update. Electron documents `autoUpdater`/Squirrel-based update flows and a packaged-app guard. The implementation decision must specify who controls release metadata/artifacts and CI signing credentials before enabling updates. [Updating Applications](https://www.electronjs.org/docs/latest/tutorial/updates)

Also review Electron fuses during packaging—especially disable `runAsNode` and `nodeCliInspect` unless a demonstrated requirement exists. Electron identifies these as command-line behaviors that can permit command execution through the app. [Security](https://www.electronjs.org/docs/latest/tutorial/security)

## Acceptance checks for the implementation plan

- An automated packaged-build check asserts every renderer has `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, and no disabled web security/insecure-content/experimental flags.
- A bridge contract test proves `window.loom` has no raw IPC/Node/Electron APIs and that malformed, cross-project, stale, unexpected-window, and unexpected-sender requests fail closed.
- An integration test proves Pi/Git receives a fixed executable, argv array, controlled `cwd` inside the allocated worktree, and no secret leaks to renderer/backend/log output.
- A path-security test covers traversal, symlinks, changed project roots, a worktree outside its project, and cancelled/restarted sessions.
- Credential tests prove plaintext is absent from renderer, backend requests, local structured logs, and normal persistence; Linux `basic_text` blocks credential persistence.
- A browser-window/security test proves navigation, popups, permissions, and external-open attempts are denied unless the concrete allowlist and a user action permit them.
