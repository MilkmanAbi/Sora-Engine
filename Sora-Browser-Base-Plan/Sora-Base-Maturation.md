# Sora Base - Maturation Guide

Version 0.1. The guide for turning the B0-B4 skeleton into a mature, expandable browser base.
Companion to `Sora-Base-Plan.md`. Worked through slowly, over sessions.

---

## The shape we're building toward

```
        Electron  ( Chromium renderer processes + Node main process )
                                   │
        ┌──────────────────────────────────────────────────────────┐
        │  ENGINE CORE  (main process, this is "the base")          │
        │  Profiles · Spaces · Tabs · Sessions/Storage · Layout ·   │
        │  Hibernation · Bookmarks · Settings · Sync · Downloads ·  │
        │  History · Permissions · ExtensionRegistry                │
        └──────────────────────────────────────────────────────────┘
                                   │
        ══════ STABLE VERSIONED ABI ══════  (src/shared: SoraApi + contract version + capabilities)
                                   │
        ┌──────────────────────────────────────────────────────────┐
        │  UI SHELL  (renderer)   harness now → PaperDesign chrome  │
        └──────────────────────────────────────────────────────────┘
                                   │
        ══════ EXTENSION CONTRIBUTION ABI ══════  (native GUI-mod extensions)
                                   │
                              Extensions (later)
```

The point of the two ABI seams: the engine below and the UI/extensions above can be upgraded
**violently and independently** as long as the seam holds. Bump the contract major, let the shell
negotiate capabilities, degrade gracefully. This is the opposite of a monolith where a change
anywhere breaks everything.

Two distinct extension tracks, deliberately separated:
- **Sora native extensions** — modify Sora's own UI (sidebar items, commands, themes, toolbar). Sora's ABI. Foundation started now.
- **Chrome web extensions** — modify page content (uBlock, etc). Via `electron-chrome-extensions`, needs Electron 35+. Later phase, own track.

---

## Workstreams (each matures independently against the ABI)

### 1. Versioned ABI + capabilities  ← spine, do first
- `CONTRACT_VERSION` (major.minor). A `hello` handshake: UI asks, engine returns `{ version, capabilities }`.
- `Capabilities` = a flat record of feature flags (`sync`, `hibernation`, `bookmarks`, `nativeExtensions`, ...). UI reads it and only shows what exists. New engine build can add capabilities without breaking an old shell; a major bump signals breaking changes the shell must handle.

### 2. Profiles  ← identity/data backbone
- A `Profile` = { id, name, color, dataDir }. Owns its own Settings, Bookmarks, Spaces, session-restore data, all under its dataDir.
- Sessions are keyed `persist:p:<profileId>:s:<spaceId>` → cookies/IndexedDB/cache/service-workers isolated per (profile, space), the modern-browser way, for free from Chromium.
- One active profile per window for now. Live profile switching is basic/experimental until the shell exists.

### 3. Storage maturation
- Persistent partitions already give durable HTTP cache, IndexedDB, Cache Storage, service workers, localStorage — this is what "multi-file caching like real browsers" *is* in Chromium; we get it by using `persist:` sessions rooted per profile.
- `StorageManager`: report cache size, clear cache, clear storage data — the foundation for a real "clear browsing data" settings pane.

### 4. Settings
- Typed `SoraSettings`, persisted per profile, streamed in state. Includes the real toggles:
  - `searchEngine` (template string; DDG default, Google/others swappable later — links preloaded).
  - `sessionRestore` (default **off** — owner hates it, friends love it, so it's a switch).
  - `tabPolicy`: `'awake'` (never sleep, background throttling off — undisrupted network, costs RAM) vs `'sleep'` (hibernate inactive tabs, throttle — costs latency, saves RAM).
  - `hibernateAfterMinutes`.

### 5. Real-browser URL handling
- `resolveInput` must handle: full schemes, `localhost:8000` and `127.0.0.1:port` → **http**, private IPs, `*.local`/`.test`, bare paths, `file://`, and search fallback. Unit-tested, no guessing.

### 6. Hibernation + keep-awake
- `HibernationManager` drives policy: `'sleep'` discards inactive tabs past a threshold (destroy the `WebContentsView`, keep a light record: url/title/favicon/scroll), rematerialize on activate; `'awake'` never discards and sets `setBackgroundThrottling(false)`.
- Discarding a live renderer is how you actually reclaim memory in Chromium — there's no magic "compress"; discard-to-record is the real mechanism, same as Chrome Memory Saver / Edge sleeping tabs.

### 7. Session restore (optional)
- If enabled, persist open tabs + spaces + active + layout per profile on quit/autosave; restore on launch. Off by default.

### 8. Bookmarks + import
- Bookmark/folder tree, persisted per profile. Import the **Netscape bookmark HTML** format (the universal export from Chrome/Edge/Firefox/Safari). Pure parser, unit-tested.

### 9. Sync (clean, last-write-wins)
- `SyncableSnapshot` = the portable subset (bookmarks + settings, later reading list). `SyncAdapter` interface: `pull()` / `push()`.
- `LocalFolderSync` — a real, working adapter (syncs via a shared folder; also the test harness for the engine).
- `GoogleDriveSync` — adapter skeleton targeting Drive `appDataFolder`; needs the owner's OAuth client — credentials are never hardcoded and never entered by the assistant.
- `SyncEngine` — last-write-wins by `updatedAt`. Deliberately simple and predictable (the anti-"wonky server" design): no operational-transform, no conflict UI, newest wins per snapshot. Manual "sync now" + interval.

### 10. Native extension foundation
- `SoraExtensionManifest` with `contributes` (sidebar items, commands, toolbar buttons, themes) — the contribution ABI.
- `ExtensionRegistry` loads/enables manifests and surfaces their contributions in `BrowserState`, so the shell can already render extension-contributed UI. No sandboxed execution yet — foundation only.

### 11. Later (own docs when reached)
Downloads, history (→ SQLite when search is slow), permission manager UI, Chrome web extensions (Electron 35+ bump), then the PaperDesign chrome replacing the harness.

---

## Invariants that must never break while maturing

- The engine never imports from the renderer. One-way dependency.
- Everything the UI can do crosses `src/shared` (typed). No stray `ipcMain.handle`.
- The UI holds no authoritative state — it renders `BrowserState` snapshots.
- Untrusted tab content stays sandboxed + isolated + Node-free, always.
- Adding engine features adds capabilities; it does not silently change existing contract shapes. Breaking shape changes bump the contract major.

---

## This session's scope (v0.2 of the base)

Fully implemented + tested: versioned ABI/capabilities, Profiles + per-profile data rooting, Settings
with the real toggles, rewritten `resolveInput` (+ unit tests), Hibernation + keep-awake, optional
session restore, Bookmarks + Netscape import (+ unit test), Sync engine + LocalFolder adapter (+ unit
test), StorageManager, native ExtensionRegistry foundation.

Staged (clean ABI, impl later): GoogleDrive sync (needs owner OAuth), Chrome web extensions (needs
Electron 35+), downloads/history/permission-UI, the real chrome.
