# Desktop → web capability map

The full mapping table behind Stage 2 of the skill. For each desktop capability: the web equivalent (if any), its support status, and the recommended strategy. "Fate" values: **keep** (web equivalent is adequate), **move** (relocate server-side), **degrade** (weaker but acceptable), **cut** (drop it, tell users).

Support labels (mid-2026): **universal** = all evergreen browsers; **chromium** = Chrome/Edge only, no Safari/Firefox; **partial** = shipped but uneven. Verify anything load-bearing on MDN / caniuse at build time rather than trusting this file's snapshot.

## Filesystem

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Read/write arbitrary paths (`fs`, `std::fs`) | None. Closest: File System Access API (`showOpenFilePicker`, `showDirectoryPicker`) with per-session user grants | chromium | Usually the decisive row. **Move** file processing server-side, or **degrade** to picker-mediated access. If the product IS a local-files tool (editors, media managers), consider keeping a desktop build alongside the web app instead of forcing the port. |
| App-private data dir (`userData`) | **OPFS** (Origin Private File System): private, per-origin, fast, works in workers | universal | **Keep.** The natural home for local-first app state. Invisible to the user as files; pair with an export flow. |
| Drag a file/folder into the app | DataTransfer + `webkitGetAsEntry` / `getAsFileSystemHandle` for directories | universal (files), partial (directories) | **Keep** for files; treat directory drops as progressive enhancement. |
| Save a file to disk | `showSaveFilePicker` (chromium) or classic `<a download>` blob fallback | universal via fallback | **Keep** with the two-tier approach. |
| Watch a folder for changes (`chokidar`, FSEvents) | None for arbitrary paths. `FileSystemObserver` exists for handles you already hold | chromium, experimental | **Move** (server watches server data) or **cut**. A local companion agent is the only true equivalent. |
| File associations / "Open with" | File Handling API, installed PWA only | chromium | **Degrade** or **cut**. |

## Storage and databases

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Embedded SQLite (better-sqlite3, rusqlite, Core Data's store) | **wa-sqlite / sql.js** (SQLite compiled to WASM) persisting to OPFS; or the official SQLite WASM build | universal | **Keep** for local-first. Run it in a worker; OPFS sync-access handles make it fast. Schema and migration files usually port unchanged, which is a huge head start. |
| Same, but the goal is multi-device | Server database (Postgres etc.) behind your API | n/a | **Move.** Reuse the SQLite schema as the starting point. Add per-user scoping to every table on day one. |
| Key-value settings (electron-store, NSUserDefaults, plists) | `localStorage` for trivial+synchronous; IndexedDB (Dexie/idb wrapper) for anything structured; or the user's account record when server-backed | universal | **Keep/move.** Do not put multi-KB blobs in `localStorage`. |
| Large binary/blob caches | Cache API or OPFS | universal | **Keep.** Mind eviction: storage is best-effort unless `navigator.storage.persist()` is granted. |
| "The user's data is safe on their disk" assumption | Browser storage is evictable and per-browser-profile | n/a | Local-first ports MUST ship export/backup (file download or optional account sync). Silent eviction of years of data is the local-first port's worst failure mode. |

## Processes and system access

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Spawn CLIs / child processes (`child_process`, node-pty, Tauri sidecars) | None in the browser | n/a | **Move** to server-side jobs (queue + worker; stream output over SSE/WebSocket), or a **local companion agent** the web app connects to. WebContainers/WASM cover narrow cases (Node-in-browser sandboxes), not arbitrary binaries. |
| Interactive terminal (node-pty + xterm.js) | xterm.js survives as the frontend; the pty must live on a server or companion, bridged over WebSocket | universal (frontend) | **Move.** This exact split (xterm.js in browser, pty daemon behind WS) is a well-trodden pattern; the renderer code often ports nearly unchanged. |
| Shell/env access (PATH, `$SHELL`, env harvesting) | None | n/a | **Move** or **cut**. Env-dependent behavior belongs to whatever machine now runs the work. |
| Native modules (.node addons, FFI) | WASM builds where they exist (sqlite yes; many others no) | varies | Check each module for a WASM sibling; otherwise **move** the capability server-side. |
| Reading system info (hostname, other apps, hardware details) | Very limited (`navigator.*` basics) | partial | **Cut** in most cases; fingerprint-adjacent APIs are deliberately restricted. |

## OS integration

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| System tray / menu bar extra | None | n/a | **Cut.** If ambient presence is core to the product, that is an argument for keeping a small desktop companion, not for faking it. |
| Global keyboard shortcuts (work when app unfocused) | None (in-page shortcuts only) | n/a | **Degrade** to in-app shortcuts. Avoid browser-reserved combos (Cmd/Ctrl+W/T/N/Q). |
| Native OS menus | In-app menu components | n/a | **Degrade.** Use accessible menu primitives (see the base-ui-primitives skill) rather than hand-rolled divs. |
| Desktop notifications | Notifications API (page open) + **Push API** via service worker (page closed) | universal; push needs permission + (iOS) installed PWA | **Keep**, but re-earn the permission: browsers punish premature prompts. Ask in context, after value is shown. |
| Launch at login / background running | Periodic Background Sync is weak and chromium-only | partial | **Cut**; background work moves to the server (cron, queues). |
| Protocol handler (`myapp://`) | Real URLs; `registerProtocolHandler` for `web+` schemes | universal (URLs) | **Keep, upgraded.** Plain URLs beat custom schemes; design routes deliberately. |
| Clipboard (read/write, images) | Async Clipboard API | universal for text; images partial (Safari lags on formats) | **Keep**; write goes through on user gesture, read is permission-gated. |
| Drag-and-drop OUT to other apps | Very limited | partial | Usually **cut**; offer download instead. |
| Dock/taskbar badge | Badging API, installed PWA | partial | **Degrade** (title-prefix fallback: `(3) Inbox`). |

## Windowing and lifecycle

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Multi-window with programmatic placement | Tabs; `window.open`; Window Management API for multi-screen placement | partial | **Degrade.** Re-think as routes + tabs; popup-blocker rules apply to `window.open`. |
| Always-on-top / picture-in-picture panels | Document Picture-in-Picture | chromium | **Degrade** or **cut**. |
| "Quit is a feature" ordered teardown | `visibilitychange`/`pagehide` + `sendBeacon`/`fetch keepalive` for last-gasp writes | universal | Redesign: persist continuously (web apps can die at any moment without events firing), never batch critical writes for exit. |
| Single-instance lock | BroadcastChannel / Web Locks to coordinate tabs | universal | Same problem, new shape: N tabs instead of N processes. Elect a leader with Web Locks if only one tab may do a job (e.g. hold a WebSocket). |
| Crash recovery / session restore | Routes + persisted state make refresh the recovery story | universal | **Keep**: if every state is URL-addressable and stores persist, refresh IS recovery. |

## Hardware and media

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Serial / USB / HID / Bluetooth | Web Serial, WebUSB, WebHID, Web Bluetooth | chromium | **Keep** only if Chromium-only is acceptable for that feature; otherwise companion agent. |
| Camera / microphone / screen capture | getUserMedia / getDisplayMedia | universal | **Keep.** Screen capture excludes system audio on most platforms. |
| GPU compute / heavy rendering | WebGPU / WebGL / WASM(+SIMD, threads) | WebGPU partial, rest universal | **Keep** with a fallback ladder; profile before promising parity. |
| Printing | `window.print()` + print CSS | universal | **Degrade**; fine for documents, weak for silent/label printing. |

## Secrets, auth, security

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Keychain / safeStorage / DPAPI | None client-side. Server holds secrets; browser holds only a session | n/a | **Move.** Any key shipped to the client is public. Third-party API calls that needed the key become proxy routes. |
| "OS login = auth" | Sessions/OAuth; passkeys (WebAuthn) are the closest to the old invisible-auth feel | universal | **Move.** Use a proven auth library/provider; never hand-roll. |
| IPC channel allowlists / Tauri allowlist | Route-level authorization + CSP | universal | The permissions *thinking* survives; it relocates to the server and the CSP header. |
| Code signing / notarization | TLS + Subresource Integrity + your deploy pipeline's integrity | universal | Disappears as a user-facing concern; supply-chain discipline moves to CI. |

## Distribution and updates

| Desktop | Web equivalent | Support | Strategy |
|---|---|---|---|
| Installer + auto-updater (electron-updater, Sparkle) | Deploys. Service worker controls cached-version rollover | universal | **Keep (upgraded)**, but the failure class survives: a bad service worker is a bricked auto-updater. Version the SW, test that deploys reach open tabs, keep a kill-switch SW ready. |
| Release channels (alpha/stable) | Preview deploys, feature flags, gradual rollouts | universal | **Keep (upgraded).** |
| Offline-first operation | Service worker precache + runtime caching; background sync for queued writes (partial support) | universal core | **Degrade** honestly: define which flows work offline, test them, and fail the rest with a clear message. |
| "Works without our servers existing" | Static hosting + OPFS gets close; true peer-to-peer sync does not | n/a | Local-first SPA on static hosting is the honest equivalent; name the residual dependency. |

## Reading this map during Stage 2

1. List the desktop app's features (not APIs; features users would name).
2. For each, find its rows here and pick a fate: keep / move / degrade / cut.
3. Anything landing in "move" implies server scope; three or more "move" rows usually means the Stage 1 shape is server-backed or split, not a thin port. Revisit Stage 1 if so.
4. The finished matrix is the port's contract and the Stage 7 acceptance checklist. Cuts get announced to users as decisions.
