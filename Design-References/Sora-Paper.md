# Sora
### A browser, built on PaperDesign.

Version 0.1 — Personal specification · extends `PaperDesign.md`

---

## 0. What this document is

This is not a second design system. Sora is a **PaperDesign application** — everything in `PaperDesign.md` (the grid, the two-shape rule, the density modes, the color *method*, the motion rules) still applies. This document only covers what's specific to being a browser: the sidebar, spaces, tabs, the address bar, reader mode, split view, and the handful of places Sora deliberately bends a PaperDesign rule on purpose.

Where this document is silent, defer to `PaperDesign.md`. Where the two disagree, this document wins for anything browser-specific — that's what it's for.

### 0.1 The one-line pitch

A browser should feel like paper on a desk: quiet, tactile, and out of the way until you reach for it. Chrome stays hairline-thin. Content is the loudest thing on screen, on purpose, every time.

### 0.2 Where Sora deliberately departs from strict PaperDesign

Two intentional deviations, both kept on a short leash:

1. **Radius runs slightly warmer than the 4px ceiling** — sidebar rows, the address bar container, and card-like surfaces sit closer to **8px**. This is a conscious choice for a browser specifically: a browser is handled constantly, all day, and a hair more softness reads as "worn-in tool" rather than "engineering instrument." It is *not* an invitation to keep climbing — 8px is Sora's ceiling the same way 4px is PaperDesign's. See §3.
2. **Workspace identity color is exempt from the one-accent rule** — each Space gets its own identity color (a dot, a rail avatar), and those can run at full, distinct saturation across a single screen. This is the same carve-out PaperDesign already grants favicons: identity markers for *other, separate things* (a favicon, a workspace) aren't semantic UI state, so they don't compete with the one-accent rule the way tag/label colors would. See §5.3.

Everything else — grid, motion, typography method, the color *method*, chrome/content separation — is inherited unchanged.

---

## 1. Color — nothing dictated, both modes are first-class

PaperDesign already refuses to mandate a palette (`PaperDesign.md` §6). Sora goes one step further and makes this a *product* requirement, not just a design-system one:

- **Light and dark are equal citizens, shipped together, from day one.** Neither is the "real" theme with the other bolted on. Every token in §2 is defined as a pair, not a default-plus-inversion.
- **Accent color is a user preference, not a brand decision.** Sora ships with a sensible default accent, but the settings surface for changing it is not buried — it's a first-run-visible choice, the same weight as picking light or dark.
- **Sora itself has no house hue.** The reference screenshots in this project use a warm cream light mode with a green accent — that's one user's configuration, not the product's identity. Don't hardcode it anywhere that a theme value should sit instead.
- **Workspace/Space colors are independently user-assignable** per §0.2(2) and §5.3 — a user should be able to make "Work" green and "Personal" orange without that touching their global accent color at all. These are two separate color systems (product accent vs. workspace identity) that happen to both be user-chosen.

The *method* from `PaperDesign.md` §6.2–6.3 still governs both modes: one undertone, four neutral steps, one muted accent with duller companions. What Sora adds is simply: ship both directions of that method by default, and expose the accent and workspace-identity choices to the user rather than fixing them at design time.

---

## 2. Chrome tokens — Sora-specific surfaces

These extend (don't replace) the base `surface-*`/`ink-*` tokens from `PaperDesign.md` §6.2. Both light and dark are defined side by side, on purpose, so neither is ever an afterthought derivation of the other.

| Token | Role | Light | Dark |
|---|---|---|---|
| `chrome-bg` | Window frame, sidebar background | `surface-canvas` | `surface-canvas` (dark scale) |
| `chrome-panel` | Sidebar row surfaces, toolbar button rest state | `surface-raised` | `surface-raised` (dark scale) |
| `chrome-active` | Selected tab/row fill | `accent-primary` at ~12% opacity over `chrome-panel` | same relationship, dark scale |
| `content-frame` | The border/inset around a loaded page or reader view | 1px `ink-tertiary` hairline | 1px `ink-tertiary` hairline (dark scale) |
| `url-pill` | Address bar fill | `surface-sunken` | `surface-sunken` (dark scale) |

No new hues here — this table exists to name *roles*, not introduce colors outside the method in `PaperDesign.md` §6.

---

## 3. Shape — Sora's 8px ceiling

Per §0.2(1): Sora uses **one radius token, `radius-chrome` = 8px**, in place of PaperDesign's stricter `radius-sm` (4px), for every rectangular chrome surface — sidebar rows, the address bar container, panel edges, hero/media placeholders in reader view.

- `radius-pill` (bars — the address bar itself, search fields) is unchanged from `PaperDesign.md` §5.1 — still a true capsule, `height / 2`.
- The **beak** shape (`PaperDesign.md` §5.1a) is unchanged and still reserved solely for tooltips/context-menu anchors.
- Content rendered *inside* a page (§4) is not bound by `radius-chrome` at all — that's the page's own design, not Sora's. See §4.

**Hold the line at 8px.** If a future draft wants softer, that's a deliberate new decision to document here, not a drift.

---

## 4. Chrome vs. content — the load-bearing rule for a browser

`PaperDesign.md` §4.1 exists for exactly this product. Restated at browser-specific granularity:

- **Chrome** = the sidebar, tab rows, address bar, toolbar, window frame, settings, reader-mode furniture (byline, "N min read," "saved to Reader"). Always Sora's own tokens (§2), always Sora's own type (§6), always `radius-chrome`.
- **Content** = whatever's loaded in the tab — a live webpage, a PDF, a reader-mode article body. This is **not** styled by Sora's tokens. A page's own fonts, colors, corner radii, and layout are the page's business. Sora's only job on the content side is the `content-frame` hairline that marks where chrome ends and the page begins.
- **Reader mode is a special, partial case.** The furniture around a reader-mode article (byline, meta line, "Less chrome, more page" section labels if editorial, image placeholders) is chrome, styled in Sora's own type and tokens. The article's *own* body copy, headline, and any author-supplied imagery are content, and are allowed to run past Sora's type-scale ceiling — this is why a reader-mode headline set well above `type-h1` (as in the Inbox mockup) is correct, not a violation. It's editorial content wearing Sora's reading furniture, not Sora's own UI speaking at that size.
- **Never let a page (or a reader-mode article) render a fake button, fake menu, or fake toolbar that could be mistaken for real Sora chrome.** This is the actual security/trust reason the line exists, not just a style preference.

---

## 5. The Sidebar

The single most distinctive piece of Sora's chrome, and the part most worth getting exactly right.

### 5.1 Structure

Three collapsible groups, top to bottom: **Spaces** (Work / Personal / Research, or whatever the user names them), then within each space, its open tabs as rows. A persistent **"New tab"** row sits at the bottom of the list, outside any space group. An **account switcher** anchors the very bottom of the sidebar, outside the scrollable tab area entirely — it's chrome-of-chrome, always reachable, never scrolled away.

- Each row = one PaperDesign grid unit tall in Compact density (§13 of the base spec), using a single-letter/favicon avatar + label + close affordance, exactly per the mockup.
- **Tabs vs. New Tab in the browser vs. Apps** are two different affordance types living in the same sidebar shell — "Tabs" (session-scoped, closable, reorderable) and "Apps" (pinned, persistent, not part of a space's unread count) are visually distinguished by the top-level tab control in the sidebar header, not by two separate sidebars. Keep that one switcher; don't fork the sidebar into two competing navigation surfaces.

### 5.2 Space identity

Each Space gets: one identity color (dot, per §0.2(2)/§5.3) and an unread-count badge in `type-caption`. The identity color is the *only* place in the sidebar where a hue outside the accent/neutral system appears — tab rows themselves stay neutral (`chrome-panel`/`chrome-active`), so the space color reads as a category label, not decoration bleeding into every row.

### 5.3 Why workspace color is exempt from "one accent per screen"

`PaperDesign.md` §6.4 caps a screen to one accent because competing saturated colors erode hierarchy. Workspace identity colors don't compete with the accent for the same job — they're doing *categorization* (which space is this tab in?), the same functional role a favicon plays, not *action-signaling* (what should I click?). A user can have a green Work space, an orange Personal space, and a blue Research space, all visible at once, and the single accent color (used for the active/selected state, links, primary buttons) still reads as the one thing that means "do this" — because it's a completely different visual job.

**The line not to cross:** workspace colors must never be reused as a tag/label/semantic-state color elsewhere in the product (§6.3 of the base spec still governs those). If Work is green, that green shouldn't also start meaning "success" in a toast notification three screens later — that would blur the two systems this section deliberately keeps separate.

### 5.4 Collapse behavior

The sidebar is a **panel** per `PaperDesign.md` §4 (persistent, toggleable, grid-aligned when open) — never an overlay. Collapsing it:

- Slides along its docked edge per the base spec's motion rules (§12.1) — Macro duration (260ms) given its size and travel distance.
- Follows the nav-list reflow pattern from `PaperDesign.md` §3.2: labels hide before icons shrink, never the reverse.
- Never dims or scrims the content behind it, since it's not an overlay — content simply gets wider as the sidebar gives up its column.

---

## 6. Typography — the three-voice system, applied

`PaperDesign.md` §7.1 recommends a contrasting-but-cohesive pairing over strict uniformity. Sora is the reference implementation of that recommendation:

| Voice | Used for | Notes |
|---|---|---|
| **Serif** (headline) | Reader-mode article headlines, section titles inside long-form content | Content-side, not chrome — see §4. Set well above the base spec's `type-h1` ceiling when it's genuinely editorial content, per §4's reader-mode carve-out. |
| **Sans** (UI) | Every chrome element without exception: sidebar rows, tab labels, address bar, buttons, settings | Stays inside `PaperDesign.md` §7.2's scale exactly — chrome never borrows the serif's size latitude. |
| **Monospace** (meta) | URLs, the `mail.sora` source line, "6 min read," timestamps, keyboard shortcuts (`⌘K`) | This is Sora's signature move — mono for anything that's *data about the content* rather than the content or the controls. It's what gives the reader-mode meta line and the address bar their quiet, technical, "this is a tool, not a magazine" feel against the warmer serif headline. |

Three voices, per the base spec's ceiling (§7.1) — hold it there.

---

## 7. Components specific to Sora

### 7.1 Address bar

A true pill (`radius-pill`, per `PaperDesign.md` §5.2's height math) — this is the one chrome element that should feel most like a PaperDesign "bar," not a "container." Contents, left to right: security/lock glyph, URL in monospace, right-aligned actions (star/bookmark, split-view toggle, overflow menu). Height follows the base spec's bar-height formula against whatever type size the URL text uses — don't let it inflate past that math just because it's the most-used control in the window.

### 7.2 Split view

Split view is a **layout state of the content area**, not an overlay and not a separate mode with its own chrome — the sidebar, address bar, and toolbar stay exactly where they are; the content region divides into grid-aligned columns per `PaperDesign.md` §2.3. Each pane gets its own `content-frame` hairline (§4) so it's unambiguous where one page ends and the next begins. Triggered by one keystroke, per the mockup's own copy ("split view is one keystroke, not a menu dive") — that line is a genuinely good statement of principle 5 (workflow beats decoration) from the base spec and is worth keeping as an actual design commitment, not just copy.

### 7.3 Reader mode

Reader mode is a **content transformation**, not a new chrome surface — the sidebar and address bar persist exactly as in normal browsing (the URL bar still shows the real page URL). What changes is the content region: the page's own styling is stripped and replaced with Sora's reading furniture (byline, meta line, hero image placeholder) in the three-voice system from §6, with the article body itself rendered in a comfortable reading measure (roughly 60–75 characters per line — this is the one place a fixed pixel width matters more than a grid-column count, since line length is a readability constraint, not a layout one).

### 7.4 Toolbar icon row

Standard PaperDesign icon-row reflow (`PaperDesign.md` §3.2) applies directly: icons compress their spacing before anything overflows, and anything that does overflow lands in a docked menu, never a popover overlay. Icon-only controls here (theme toggle, downloads, search) are permitted without labels *only* because they're extremely common, well-understood browser conventions — this is principle 8 from the base spec (spend distinctiveness on a few moments, stay familiar everywhere else) in direct practice. Don't extend the same icon-only leniency to a novel, Sora-specific control the user hasn't seen in another browser before — that one needs a label until it's proven itself familiar.

**One functional note independent of visual spec:** a theme control should represent a single toggle with one icon showing the mode you'd switch *to* — not a sun and a moon both visible simultaneously, which reads as two controls instead of one.

---

## 8. Anti-patterns specific to Sora

- ❌ Hardcoding the cream/green combination from early mockups anywhere in code as a default users can't change — that configuration is a demo, not the product's identity (§1).
- ❌ Letting a workspace identity color leak into the tag/semantic-color system, or vice versa (§5.3).
- ❌ Styling loaded page content or reader-mode article bodies with Sora's own chrome tokens (§4) — content keeps its own design; Sora only frames it.
- ❌ Radius creeping past 8px on any chrome surface, or dropping below it back toward the base spec's 4px inconsistently across components (§3) — pick 8px and hold the whole product to it.
- ❌ A second, competing tab-management surface alongside the sidebar (§5.1) — one navigation spine, not two.
- ❌ Split view or reader mode reflowing the sidebar/address bar chrome — those are content-region states only (§7.2, §7.3).
- ❌ A theme toggle showing both light and dark icons at once instead of one icon representing the available action (§7.4).

---

## 9. Quick Reference

```
INHERITS       PaperDesign.md — grid, motion, density modes, color METHOD, type scale
RADIUS         8px (radius-chrome) for all Sora chrome — not PaperDesign's 4px ceiling
               · pill unchanged for bars · beak unchanged for tooltip/menu anchors
COLOR          zero hardcoded hues. light + dark both first-class. accent = user choice.
               workspace/space color = separate, user-assignable, exempt from 1-accent rule
               — but never reused as a tag/semantic color (keep the two systems apart)
TYPE VOICES    serif (content headlines only) · sans (all chrome, no exceptions)
               · mono (URLs, meta, timestamps, shortcuts) — 3 voices, hold the ceiling
CHROME/CONTENT sidebar/address bar/toolbar = Sora's tokens always
               loaded pages + reader-mode article bodies = their own styling, framed only
SIDEBAR        panel, not overlay · Spaces → tabs → New tab → account switcher (anchored)
SPLIT VIEW     content-region layout state, not a new chrome mode
READER MODE    content transformation, address bar still shows the real URL
```
