# Sora - Design & Build Plan

Version 0.2 - decisions locked. Companion to `PaperDesign.md` and `Sora-Paper.md`.
A Chromium-based (Electron shell) browser. A slow passion project, not a product. Built for its makers to actually use.

---

## 0. What Sora is, and what it refuses to be

Sora is a hobbyist browser made by its author, a friend, and Claude, for the pleasure of using something built with care and shared online if anyone wants it. Success = "I use it and it works." That's the whole bar. This framing changes real engineering decisions, so it's written down first.

**The philosophy:**

- **Productivity over feature count.** Chrome and Edge bury you in Copilot, Gemini, wallet, shopping, 300 toggles nobody asked for. Sora is the opposite: a small set of features that earn their place by helping you work, and nothing shoved at you.
- **Opt-in, never opt-out.** Utility tools will exist (screenshot, text extract - the stuff every browser has). They stay off until you turn them on. Features are offered, not pushed.
- **Guaranteed workflow, forever.** The sidebar, split view, and PaperWorkspace are load-bearing and permanent. Sora will not wake up one day and delete the sidebar the way Edge did. If it ships as core workflow, it stays.
- **Respect the user.** No nagging, no dark patterns, no engagement metrics, no telemetry. The browser works for the person, not for a growth chart.
- **Slow and correct beats fast and impressive.** No rush-to-prototype, no showoff build. Do it right, over time.

**Non-goals (things Sora deliberately will not chase):**

- Not trying to be popular, big, or "the new browser."
- No AI assistant baked into the chrome, no built-in VPN, no crypto wallet, no shopping/coupon junk.
- No privacy-theatre marketing (that's a separate, honest feature discussion, not a sales pitch).
- Not trying to beat Chrome on RAM (it can't - see §5 - and won't pretend to).

**The one strategic lesson worth keeping:** Arc died in May 2025, and its CEO's post-mortem was that it was "too different, too many new things to learn, for too little reward." That is exactly what PaperDesign principle 8 already guards against. Familiar everywhere, distinctive in a few chosen moments. Sora's job is to be quietly good, not novel for its own sake.

---

## 1. Decisions locked (was: open questions)

| # | Decision | Locked value |
|---|---|---|
| 1 | Sora license | **GPL-3.0** (also clears the extension-library license cleanly, §3) |
| 2 | Platforms | **Linux + Windows + macOS, best-effort.** "If it builds, it's good." No installer/signing/notarization burden required. |
| 3 | Chrome UI stack | **React 18 + TypeScript + Vite.** TS is the key: a typed chrome-to-main IPC contract makes future UI/UX surgery safe. Preact is an approved drop-in if React feels heavy. |
| 4 | Spaces vs profiles | **1:1** - each Space is its own session partition. |
| 5 | Command palette | **Future / optional / poweruser.** Not a near-term feature. The little things matter more (§8). |
| 6 | Install-as-app | **Independent, two modes** (Light/Tauri default, Linked/Electron opt-in). Resolves the cache tension - see §6. |
| 7 | Persistence | **JSON-first for everything; SQLite added later for history only, if/when history search is actually slow.** |
| 8 | First artifact | **Running code, unhurried.** No showoff prototype. Dogfood-driven. |

---

## 2. The core architecture (load-bearing)

Electron's process model mirrors Chrome: one main process controls the app; each page renders in its own renderer process. You compose the window out of *views*. No Chromium compile, no C++.

**Use `BaseWindow`, not `BrowserWindow`.** `BrowserWindow` is now sugar for "a BaseWindow with one built-in WebContentsView." For chrome + N tabs you want raw `BaseWindow` so you can add views by hand.

```
BaseWindow  (frameless OS window, you draw the frame)
 ├─ WebContentsView  →  Sora chrome UI  (the .dc.html chrome, rebuilt as a real React app)
 └─ WebContentsView  →  tab 1  (real webpage, its own renderer + session)
    WebContentsView  →  tab 2  ... (only active/split ones attached & visible)
```

- `<webview>` tag: legacy, avoid. `BrowserView`: deprecated since Electron 30, don't start there. `WebContentsView` is the modern Chromium-Views-backed view. Performance ~1% of old BrowserView, tracks Chromium upgrades cleanly.
- Chrome renderer is *trusted* (PaperDesign §4.1) - gets Node/preload. Tab views load untrusted content: `sandbox: true`, `contextIsolation: true`, no Node. The chrome-vs-content trust boundary Sora-Paper §4 demands, enforced at the process level.
- Main process owns layout: reads the chrome's content-region rect (sidebar width, address-bar height, split state) over IPC, calls `view.setBounds()` on the active tab view(s). Split view = two tab views positioned side by side. Reader mode = a content transformation you render; address bar untouched (matches Sora-Paper §7.2/§7.3).
- **Reference to read (not blindly fork):** `samuelmaddock/electron-browser-shell` - the canonical minimal tabbed Electron browser, origin of the extension library below. Study its tab manager and view-layout; keep your own chrome.

---

## 3. Extensions

`electron-chrome-extensions` (the API shim) + `electron-chrome-web-store` (install from the Chrome Web Store). Both by Samuel Maddock.

- **Manifest V3** now works (landed via upstream Electron). Requires **Electron 35+**; use newer.
- MV2 is dead on stable Chrome since late 2024 - Sora is MV3-only, which is all the ecosystem is now.
- **Chromium base is a real advantage over Zen.** Zen (Firefox/Gecko) can't run Chrome Web Store extensions - a repeated migration complaint. Sora being Blink-based means uBlock Origin Lite, password managers, etc. just work. Free win from choosing Electron.
- **License:** `electron-chrome-extensions` is dual-licensed GPL-3.0 / paid-proprietary. Since Sora is **GPL-3.0**, this is clean - no cost, no conflict. (This is one reason the GPL-3.0 choice is convenient.)
- Extensions are opt-in, in keeping with the philosophy - Sora ships none by default; the user installs what they want.

---

## 4. Profiles, Spaces, and container isolation

Spaces (Sora-Paper §5) become a real engine feature, not just sidebar grouping.

- Each tab view uses `session.fromPartition('persist:<space-id>')`. Cookies, storage, cache, logins isolated per partition.
- **One partition per Space (1:1, locked).** "Work" and "Personal" can each hold a different Google login with no incognito juggling - the multi-account-container win, structural instead of bolted on.
- Per-Space extension sets fall out of this naturally (extensions install against a session).
- This is a *guaranteed forever* feature per §0.

---

## 5. Resource strategy (honest version)

**Electron has a memory floor. Sora will not undercut Chrome on RAM, and will never claim to.** Each tab is a renderer process with its own heap; Electron eats free memory until the OS pushes back. What you *can* do is make idle tabs cost almost nothing - which is exactly what Edge's sleeping tabs and Chrome's Memory Saver do.

Baked in from the start:

- **Tab hibernation as a core feature.** An inactive tab past a threshold has its `WebContentsView` destroyed, reduced to a lightweight record: URL, title, favicon, scroll position, form state where feasible. Re-materialize on activation. A hibernated tab costs a sidebar row and nothing else.
- Chromium already auto-discards after ~5 min idle - you want that; surface and tune it in settings rather than fight it.
- `backgroundThrottling: true` on tab views; throttle timers/rAF for hidden tabs.
- Hibernation pairs with Spaces: switching Space can hibernate the outgoing Space's tabs wholesale. A workflow feature and a memory feature at once.
- Show hibernation calmly (dimmed favicon, per PaperDesign motion/feedback) so "light" is visible, not a slogan.

This makes the honest weakness (Electron heft) into a real, visible feature. That's the right way to handle it in a project that refuses to overclaim.

---

## 6. Install-as-app: independent, two modes

Your instinct (lean site-apps in Tauri) is right - Tauri v2 uses the OS webview (WebView2/Windows, WebKitGTK/Linux, WKWebView/macOS), ships ~5-10MB, idles ~30-50MB RAM.

**Correction to the mental model:** a Tauri app is a compiled Rust binary. Electron can't spawn a Tauri window at runtime. The buildable pattern: compile **one generic `sora-app-runner` binary once**; it takes a URL + name + icon (CLI args or a tiny per-app config it reads on launch). "Install as app" compiles nothing - it writes the OS integration (`.desktop` on Linux, Start Menu shortcut on Windows, `.app` wrapper on macOS) pointing at `sora-app-runner` with that app's config. Launching the shortcut runs the light runner, not Electron.

**The cache tension, and its resolution.** You wanted site-apps to share cache with Sora for a seamless feel. Physics problem: Tauri site-apps render in the OS webview (WebKitGTK etc.); Sora renders in Electron's Chromium. Different engines, incompatible storage stores. You cannot share one cookie/cache jar between WebKitGTK and Chromium - it's a format mismatch, not a wiring gap. Forcing a shared data folder is fragile and unsupported.

So "ultra light (Tauri)" and "shares session with the browser (needs the same engine = Electron)" are in genuine tension. Resolved with **two install modes, user picks per app** - which is the opt-in ethos applied to architecture:

- **Light app (default):** Tauri runner, own session, but **all Sora site-apps share one store with each other**. Log into a service in one site-app, you're logged in across your site-apps. Tiny footprint. What most installs want.
- **Linked app (opt-in):** a chromeless **Electron** window that shares Sora's actual session partition - literally the same cookies/cache as the browser, truly seamless. Costs Electron weight. For the one or two apps you want to feel exactly like "already logged in in my browser" (e.g. Gmail).

Default lightness, opt-in seamlessness, per app. No global compromise.

Caveats to plan around:
- **WebKitGTK on Linux** lags Chromium on CSS/JS and has per-distro quirks; a page in a Light app renders slightly differently than the same page in Sora proper. Test the runner on your actual Linux target.
- **Windows 10** may need a WebView2 bootstrapper.
- Rust build friction is a one-time cost - you compile the runner once, not per app.

---

## 7. What to steal, what to avoid

| Browser | Steal | Avoid |
|---|---|---|
| **Arc** (dead May 2025, maintenance-only) | The workspace-browser idea: sidebar spine, Spaces, peek previews, silky motion | Its fatal flaw: too much novelty, too much to relearn. Your cautionary tale. |
| **Zen** (active Firefox-based Arc heir) | The concrete daily-use feature checklist (below) - your closest comparable | Interface roughness/jank; no Chrome extensions (their weakness = your strength) |
| **Edge** | Clean workflows, sleeping tabs, Collections, tasteful vertical tabs | Copilot bloat, nagging, ripping out the sidebar on users (the thing Sora promises never to do) |
| **Firefox** | Reader mode, containers, feature breadth | Dated chrome in places |
| **Safari** | Cleanliness, restraint, content-is-loudest | Closed, thin extensions |
| **Vivaldi** | Proof power users want depth | Over-customization - Sora's explicit anti-goal. A settings panel wearing a browser. |
| **Opera** | - | The omnibox nobody uses. Poweruser features are fine as opt-in, never as the headline. |

**Zen feature checklist** (parity bar for "an Arc-like people would actually switch to"): vertical sidebar + Spaces, per-Space pinned Essentials, split view (up to 4), compact/focus mode, Glance peek, per-Space container isolation, tab folders, color-coded Spaces (already have), quick keyboard Space switching with a clear current-context indicator.

---

## 8. Feature roadmap, tiered

Dependency order, roughly. Don't build a tier before the one under it works. Pick a few per tier - the point is restraint.

### Tier 0 - "it is a browser"
Navigation (back/forward/reload/stop), address bar (URL vs search resolution), per-tab WebContentsView lifecycle, new/close/reorder tab, history, downloads, find-in-page, devtools passthrough, session persistence across restart, settings that save to disk.

### Tier 1 - the guaranteed workflow spine (permanent, per §0)
Vertical sidebar with Spaces, tabs-as-rows, New Tab row, account switcher (designed). Split view as a content-region layout state. Reader mode as content transformation. Tab hibernation (§5). Per-Space session partitions (§4). Extensions (§3, opt-in). Light/dark + user accent + density modes (in spec).

### Tier 2 - the little things that are actual features (this is the soul)
Not selling points - things you reach for daily. Prioritized:

- **Tab folders** inside a Space, collapsible, drag to organize. (Zen shipped these late and it hurt - have them early.)
- **Pinned Essentials per Space** - the sites always one key away.
- **Tab hover preview** - hover a sidebar tab, see a small live/last-frame preview. Quiet, genuinely useful.
- **Glance / peek** - alt-click a link to preview in a transient overlay without leaving the page (a legit overlay per PaperDesign §4 - it's dismissible).
- **Tab grouping** - group related tabs, collapse as a unit.
- **Session snapshots** - name and save a set of tabs, restore later. Make them **exportable** (Arc's fatal gap was locking users in; portable data is both a feature and an insurance policy).
- **Auto-archive** - tabs untouched N days move to an archive list instead of rotting. Calm tab hygiene.
- **Focus/compact mode** - hide chrome to a hairline, one keystroke, to PaperDesign's motion standard.

Deferred to future/optional: command palette (poweruser, not killer), link-to-Space rules, anything omnibox-flavored.

### Tier 3 - PaperWorkspace mode
The macOS-desktop metaphor (dock, global menu bar, tabs-as-app-windows), already mocked. Your one big distinctive swing - the personality budget from PaperDesign §1.8 spent deliberately. Sequence after Tier 1 is solid. Guaranteed forever, per §0.

### Tier 4 - install-as-app
Tauri runner + OS integration + the two-mode design (§6). Self-contained; touches the Electron core barely.

### Utility tools (opt-in, table stakes, not the point)
Screenshot (full/region/full-page), text extract / copy-as-markdown, maybe a reader-to-note capture. All off by default, all toggleable. Present because every browser has them; never shoved.

### Settings surface (design once, spans tiers)
Top-menu + search, Edge-clean. Sections: search engine, Spaces/profiles, appearance (theme/accent/density/tab-layout), extensions, permissions manager (camera/mic/location/notifications per site), updates, downloads, keyboard shortcuts, installed apps, utility-tool toggles. A gridded *panel* per PaperDesign §4 - no floating cards. Mostly wiring; the visual is done.

---

## 9. Build sequencing (unhurried, dogfood-driven)

No deadline, no showoff milestone. Sequence so you're *using* it as early as possible, because dogfooding is how a for-me project stays honest.

- **A - "real tabs under your chrome."** BaseWindow + chrome view + one tab view + navigation + new/close. The moment your sidebar opens a real page, it's a browser you can start living in.
- **B - the spine.** Spaces with real session isolation, sidebar wired to real tabs, settings persisting (JSON). Now you dogfood daily.
- **C - the little things.** Folders, Essentials, hover preview, glance, grouping, hibernation, extensions, reader, split. Add them as you miss them - that's the correct order for a for-me tool.
- **D - the showpieces.** PaperWorkspace + install-as-app. Whenever. Parallelizable with your friend.

Public repo whenever you feel like it, led by the design system + a screenshot. No pressure to.

---

## 10. Stack (current mid-2026)

- **Shell:** Electron 35+ (MV3 extensions; newer is better).
- **Windowing:** `BaseWindow` + `WebContentsView`.
- **Extensions:** `electron-chrome-extensions` + `electron-chrome-web-store` (GPL-3.0, clean under Sora's license).
- **Chrome UI:** React 18 + TypeScript + Vite. `.dc.html` visuals port to JSX ~1:1; OKLCH CSS-variable tokens carry over untouched. The `support.js` template engine does NOT ship - rebuild chrome as a thin component tree. Preact approved as a lighter drop-in.
- **Fonts:** Hanken Grotesk / IBM Plex Mono / Instrument Serif, bundled (offline + privacy), not hotlinked.
- **Persistence:** JSON store for all config-shaped data (settings, Spaces, folders, Essentials, snapshots, layout). Add SQLite (better-sqlite3) for history only if/when history search is slow. No premature native modules.
- **Install-as-app runner:** Tauri v2 (Rust), one generic binary.
- **Packaging:** electron-builder if/when you want distributable builds (mature, cross-platform, auto-update). Optional given "if it builds it's good."
- **Reference to read:** `samuelmaddock/electron-browser-shell`.

---

## 11. Risks and realism

- **Electron weight is real.** Mitigated by hibernation (§5), never eliminated. Never frame Sora as lighter than Chrome. Fits the no-overclaiming ethos.
- **A browser is a forever-maintenance project** that tracks Chromium/Electron releases. That's fine for a slow passion project as long as it's scoped as one and never promises sync/infra it can't sustain. Arc died partly because a small team couldn't keep up - Sora sidesteps this by not trying to be big. **Build data export (snapshots, settings) early** so nothing strands the user if the project ever pauses. Direct lesson from Arc.
- **The novelty trap.** Every distinctive feature earns its place against "would this confuse first-time me?" PaperWorkspace is the one big swing; keep the rest familiar.
- **WebKitGTK Linux site-apps** render differently than Sora proper - test early or install-as-app surprises you.
- **macOS** looks like PaperWorkspace's inspiration but is the platform you use least; best-effort build is the right call, deep native integration (signing, real traffic-light controls) is out of scope unless you feel like it.

---

## 12. Two small spec-compliance nits to fix when the mock becomes code

- Outer window radius is 15px, some buttons 9px, in the `.dc.html`. Sora-Paper §3 sets an 8px chrome ceiling. Pick 8, hold it.
- Traffic-light dots are decorative in the mock. On a real frameless window they become actual window controls, which is legitimate (PaperDesign §15 only objects to fake ones). Wire them to real minimize/maximize/close.

---

*Nothing here needs building yet. When you want to start, Phase A is the entry point: a BaseWindow + your chrome + one real tab + navigation, in the Linux box, using your existing chrome markup as the UI. No rush.*
