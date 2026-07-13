---
name: desktop-to-web
description: "Framework for turning an existing desktop app (Electron, Tauri, or native) into a web app - identify what is actually inside the .app/.dmg/.exe, choose the target web architecture (local-first SPA, server-backed, thin port), map every desktop capability to a web equivalent or a deliberate cut, invert the security model, extract an Electron/Tauri renderer or rewrite a native UI, migrate user data, and ship. Use whenever the user wants a web version of a desktop app, mentions converting a .dmg/.app/.exe into a website or web app, unwrapping an Electron or Tauri app, 'put the desktop app in the browser', or making a desktop product shareable by URL / multi-device. Also use it to audit a half-finished desktop-to-web port."
---

# Desktop app → web app

Framework for porting an existing desktop app to the web. It is the mirror image of a web-to-desktop wrap: instead of gaining OS powers and losing URLs, you gain URLs, zero-install distribution, and instant updates, and you lose direct access to the user's machine. The whole job is managing that trade deliberately instead of discovering it one broken feature at a time.

**Scope note:** this skill assumes you own the app or have the rights to its code. Inspecting a third-party binary to identify its stack is fine; extracting someone else's app to republish it is not a porting job and is out of scope.

## Step 0 - Orient: what is actually inside the app?

Best case you have the source repo; then this step is just confirming the stack. Given only a `.dmg`, mount it (`hdiutil attach My.dmg`) and inspect the `.app` bundle:

| Marker inside `Contents/` | Stack | Port strategy |
|---|---|---|
| `Frameworks/Electron Framework.framework` + `Resources/app.asar` | **Electron** | Extraction: the renderer is already web code. `npx @electron/asar extract app.asar out/` to see it. Go to Stage 4. |
| One small Mach-O binary, `otool -L` shows WebKit, few/no frameworks | **Tauri / native webview wrapper** | Extraction: web assets are embedded in the binary or under `Resources`. Backend commands are Rust. Stage 4. |
| `Frameworks/App.framework` + `Flutter.framework` | **Flutter** | `flutter build web` exists; port, don't rewrite. Renderer caveats (canvas-based DOM) apply. |
| Qt frameworks (`QtCore`, `QtWidgets`) | **Qt** | Qt-for-WASM exists but is rarely the right call; usually a rewrite. Stage 5. |
| Plain Mach-O, nibs/storyboards, Swift/ObjC symbols, no web framework | **Native AppKit / SwiftUI** | Rewrite: the app becomes the spec. Stage 5. |

`defaults read /Volumes/My/My.app/Contents/Info.plist` gives the bundle id and often the framework generation. The same triage works on `.exe`/AppImage with different markers (e.g. `resources/app.asar` next to the binary means Electron).

**The fork that decides everything:** Electron/Tauri means the UI layer survives (extraction). Native/toolkit means the UI must be rebuilt and the desktop app serves as an executable spec (rewrite). Both paths share Stages 1-3 and 6-7.

## Stage 1 - Pick the target architecture (decide, don't drift)

First write down *why* this is moving to the web: URL shareability, zero-install onboarding, multi-device, instant updates, collaboration. If none of those apply, stop; a port with no motive produces a worse desktop app in a tab.

| What the desktop app's backend does | Target web shape |
|---|---|
| Pure local tool: all state on the user's disk, no server, no accounts | **Local-first SPA.** Static hosting, persistence in OPFS/IndexedDB, optional PWA install. No backend, no auth. Closest to the desktop feel; data now lives per-browser (plan export/backup). |
| Already talks to a remote API you control; local footprint is thin (settings, cache) | **Thin port.** Renderer to static hosting; move any bundled secrets behind server routes; settings to `localStorage` or the account. Fastest path. |
| Spawns CLIs, watches folders, reads arbitrary paths, talks to hardware | **Split app.** Web UI plus either (a) the work moves server-side, or (b) a small local companion/agent the web app talks to. Some capabilities cannot leave the machine; the capability map decides which. |
| The port's motive is multi-device / multi-user / collaboration | **Server-backed.** Real database, auth, per-user scoping, maybe sync. Biggest lift. Add local-first sync only if offline genuinely matters; sync engines are expensive to own. |

Mixed shapes are normal (thin port + one server route; local-first + optional account). What is not normal is deciding this at Stage 4 by accident.

## Stage 2 - Capability map (every feature gets a fate before any code)

Inventory every OS-touching feature of the desktop app and assign each one of four fates: **keep** (a web equivalent exists), **move** (runs server-side now), **degrade** (weaker but acceptable web version), **cut** (explicitly dropped, users told). This parity matrix is the port's contract; it becomes the acceptance checklist in Stage 7.

The full table with API names, browser-support caveats, and fallback strategies is in [references/capability-map.md](references/capability-map.md). The load-bearing rows:

| Desktop capability | Web reality |
|---|---|
| Arbitrary filesystem read/write | File System Access API is Chromium-only and permission-gated; OPFS is private per-origin storage; everything else moves server-side or through explicit file pickers. Usually the hardest section of the map. |
| Embedded SQLite | wa-sqlite / sql.js on OPFS for local-first; a server database for server-backed. IndexedDB (via a wrapper like Dexie) only for simple document/KV shapes. |
| Spawning processes / CLIs | Does not exist in the browser. Move to server jobs, or a local agent over WebSocket. This is the classic port-killer; surface it in Stage 1, not week six. |
| Tray, global shortcuts, OS menus | Cut, or degrade to in-app equivalents. Installed-PWA shortcuts are shallow. |
| Auto-update | Becomes a deploy. The failure mode moves too: service-worker stale caches are the new broken updater. |
| Keychain / safeStorage secrets | No client-side equivalent. Secrets live server-side, full stop. |
| Deep links / protocol handlers | URLs. The one row where the web version is strictly better; design real routes early. |

## Stage 3 - Invert the security model

A desktop app trusts the machine it runs on. A web app must trust nothing that arrives from the client. Retrofitting this is miserable, so flip it before porting features:

1. **Every byte shipped to the browser is public.** API keys, tokens, and privileged logic in the old renderer or main process move behind server routes that hold the secret and proxy the call.
2. **Auth appears.** The desktop app's implicit auth was the OS login. The web app needs sessions/OAuth from day one if anything is per-user. Use a boring, proven auth library; never hand-roll.
3. **Validation moves server-side.** IPC handlers could trust their caller; HTTP routes cannot. Validate and authorize on every route as if the renderer were hostile, because now it can be.
4. **Multi-tenancy by construction.** Local files were single-user automatically. Every server query now needs user scoping; one missing `WHERE user_id =` is a data leak, not a bug.
5. **CSP replaces the Electron security checklist.** Set a strict Content-Security-Policy early (no `unsafe-inline` if you can avoid it) and let it shake out inline-script habits while the diff is small. CORS: same-origin API needs nothing; cross-origin needs a deliberate allowlist, never `*` with credentials.
6. **The localhost pattern reverses.** Desktop apps run token-guarded local servers; web apps must never depend on one existing. If a local companion survives (split shape), the browser talks to it over an authenticated WebSocket with explicit user consent.

## Stage 4 - Extraction path (Electron / Tauri)

The renderer is the asset; the job is severing its Node/OS roots.

- **Inventory the bridge first.** Grep the renderer for `window.api`, `ipcRenderer.invoke`, `ipcRenderer.on`, and (Tauri) `invoke(` / `listen(` from `@tauri-apps/api`. That inventory IS the backend API spec: request/response channels become HTTP routes, event-push channels become SSE or WebSocket. Write it down as a table before touching code.
- **If the app was built with a service-interface seam** (facade classes like `DocumentService` wrapping IPC, the pattern a well-built desktop wrap uses), the port is mechanical: implement the same interfaces over `fetch`, leave the UI untouched. If not, introduce the interfaces now on the desktop side, then swap; a two-step refactor beats surgery.
- **Sort every main-process handler** into: server route (business logic, file/DB work), client code (things the browser can do itself, like clipboard), or delete (window management, tray, updater, app-lifecycle glue; roughly a third of most main processes evaporates).
- **Point the bundler at a browser target and fix the fallout.** Every `fs`/`path`/`os` import that Electron tolerated in the renderer is a seam you missed; resolve each by moving the logic server-side or deleting it. Do not "fix" these with polyfill shims; that hides real seams.
- **Routing:** desktop builds often use hash routing or query-param modes because of `file://`. Restore history routing and design real URL structure; deep-linkable state is a headline benefit of the port.
- **Delete the preload/contextBridge layer** and Tauri `tauri.conf.json` allowlists; their permissions thinking migrates into route-level authorization on the server.

## Stage 5 - Rewrite path (native app)

- **The desktop app is the executable spec.** Catalogue screens, flows, keyboard shortcuts, and empty/error states from the running app before reading any of its code.
- **Extract the data model first.** Core Data schemas, SQLite files, and plists under `~/Library/Application Support/<app>` and `~/Library/Preferences` define the real entities. Port the model, then read-only views, then mutations, then polish. Vertical slices; never all-screens-at-once.
- **Rebuild the UI on unstyled accessible primitives** rather than hand-rolling behavior: menus, dialogs, popovers, and comboboxes are where native apps quietly did accessibility for you. The `base-ui-primitives` skill in this repo (`npx skills add IBatho/skills@base-ui-primitives`) covers that layer for React; it exists precisely to pair with this stage.
- Native niceties (drag-and-drop between apps, services menu, Spotlight) go through the Stage 2 matrix like everything else; most are cuts, and naming them as cuts is what keeps the port honest.

## Stage 6 - Migrate user data (do not strand users)

Existing users have data on disk: electron-store JSON, SQLite files, documents under `userData`. The port fails socially, not technically, if that data dies.

- **Locate it:** `~/Library/Application Support/<app>/` on macOS (equivalents on Windows/Linux), plus preferences plists.
- **Pick a lane:** (a) ship one final desktop release whose job is "Export archive" + a web "Import" flow; (b) the final desktop release syncs directly to the new backend (one-time push, needs auth in the desktop app); or (c) the web app parses the raw files users drag in (works without shipping a desktop update, but you now parse every historical format).
- **Version the export format**, validate on import, and make import idempotent so a retry never duplicates data.
- **Do not sunset the desktop app until real user data has round-tripped.** Test with the largest, oldest, weirdest profile you can obtain, not a fresh install.

## Stage 7 - Recover the app feel, then ship

- **PWA layer:** manifest, icons, installability, service worker for offline/instant-load. Treat the service worker as the successor to the auto-updater, with the same failure class: version it, test that a deployed update actually reaches an open tab, and keep a kill switch (a trivial SW you can deploy to un-break caching).
- **Keyboard shortcuts:** reimplement in-app (globals are gone), avoid browser-reserved combos (Cmd/Ctrl+W, T, N), and keep the desktop app's muscle memory where possible.
- **First-load performance budget.** The desktop app amortized its 150MB at install time; the web app pays at every first visit. Code-split, measure, and set a budget before adding dependencies.
- **Ship:** static hosting or a server platform per Stage 1, CI deploy on merge, and the Stage 2 parity matrix as the acceptance checklist. Announce the cuts to users as decisions, not apologies.

## Definition of done

- [ ] Parity matrix exists; every desktop feature has an explicit keep/move/degrade/cut, and the shipped app matches it.
- [ ] `grep` of the production client bundle finds no API keys, tokens, or privileged endpoints.
- [ ] If server-backed: two test accounts cannot see each other's data (actually tested, not assumed).
- [ ] A real user's desktop data imports successfully, twice (idempotency).
- [ ] Offline behavior is deliberate: works offline (tested) or fails with a clear message; never a white screen.
- [ ] Deploying a new version reaches an already-open tab (service-worker update path tested).
- [ ] Core app states are deep-linkable by URL and survive refresh; being on the web was the point.

## Reference

- [references/capability-map.md](references/capability-map.md) - the full desktop-to-web capability table: per-API web equivalents, browser support status, and fallback strategies, grouped by domain (filesystem, storage, processes, OS integration, windowing, hardware, media, secrets, distribution).
