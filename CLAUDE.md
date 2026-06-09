# Figma Prototype Presenter — context for LLMs

This repo is a **single-file HTML presenter** that wraps a **live, interactive Figma
prototype** (an `embed.figma.com` iframe) in a clean Apple-style shell with a zoom
control, fullscreen, and a minimal bottom dock.

Read this whole file before changing `index.html`. The behaviors below were each
discovered by fixing a concrete bug; reverting any of them brings the bug back.

This is deployed publicly on GitHub Pages, so it ships with NO baked-in prototype
link (`DEFAULT_LINK = ""`): a fresh visitor sees the empty state — the SAME top
link panel as the normal "Link" editor (shown open), plus a centered placeholder-
style hint and a dashed arrow pointing up at the panel (`.welcome-center`). Pasting
a link removes `.no-link` and reveals the presenter.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The presenter / app. Self-contained: HTML + CSS + JS, no build, no deps. Served at the Pages root. |
| `README.md` | Public-facing project description. |
| `figma-prototype-zoom-original.html` | Frozen copy of the very first version (green glass UI). **Git-ignored** — it contains a real prototype link + share token, so it stays local and is never pushed. |
| `.claude/launch.json` | Local static server (`python3 -m http.server 8753`) for previewing. Git-ignored. |

## Non-negotiable product decisions

1. **Live interactive embed only — never a PNG / static export.** A PNG loses
   interactivity, animations, hover/click states and prototype navigation. The
   whole point is a real, clickable prototype. Do not propose static export.
2. **It must feel like a real responsive website**: header pinned to the top,
   content reflows to the viewport, page background fills the space, consistent
   size across pages (no per-page jump).
3. **Minimal Apple-style UI**: flat translucent glass dock, no gradients, no
   borders/outlines, system font, accent = Apple system blue `#0a84ff`. Stage
   background is a light neutral (`#f5f6f8`), never near-black.

## The Figma embed URL — how it works and how to build it

This is the part that matters most when adding/replacing prototype links.

### Two URLs are kept in the JS

- `ORIGINAL_PROTO_URL` → `https://www.figma.com/proto/...` — opened by the "Figma ↗"
  link in a new tab (the normal Figma player).
- `EMBED_URL` → `https://embed.figma.com/proto/...` — loaded into the `<iframe>`.

### Required query params on `EMBED_URL` (THIS IS THE CRITICAL PART)

| Param | Value | Why |
|-------|-------|-----|
| `scaling` | **`contain`** | **Must be `contain`.** With `fit-width`, when the viewport is much wider than the design (i.e. in fullscreen) Figma re-fits a freshly-visited frame in two visible passes — it renders it small, then scales it up → a "small → big" jump on every first navigation. `contain` does not do this. This was verified against the user's reference link, which uses `contain` and has no jump. |
| `content-scaling` | **`responsive`** | The "real website" mode: the design reflows to the viewport per its constraints/auto-layout. `fixed` instead scales the whole frame as one block, which makes pages jump bigger/smaller when navigating between differently-sized frames. |
| `footer` | `false` | Hide Figma's embed footer. |
| `viewport-controls` | `false` | Hide Figma's zoom/viewport controls. |
| `device-frame` | `false` | No device bezel. |
| `embed-host` | any short string (e.g. `local-prototype-zoom`) | Figma requires an embed-host. |
| `node-id`, `page-id`, `starting-point-node-id`, `t` | copy from the share link | Identify the file, starting frame and share token. |
| `viewport` | optional | Saved camera (x,y,zoom). Harmless to keep; `contain` overrides the fit anyway. |

### Recipe: turn a Figma share/proto link into a correct `EMBED_URL`

Given a link copied from Figma's Share → Prototype, e.g.:

```
https://www.figma.com/proto/<FILE_KEY>/<FileName>?node-id=382-82048&p=f&viewport=630,284,0.26&t=<TOKEN>&scaling=contain&content-scaling=responsive&starting-point-node-id=382:81671&page-id=382:81670
```

Do this:

1. **Change the host**: `www.figma.com/proto` → `embed.figma.com/proto`. Keep
   `<FILE_KEY>/<FileName>` exactly.
2. **Force the scaling params** (overwrite whatever the share link has):
   `scaling=contain` and `content-scaling=responsive`.
3. **Append the embed display params**:
   `&footer=false&viewport-controls=false&device-frame=false&embed-host=local-prototype-zoom`.
4. **Keep** `node-id`, `page-id`, `starting-point-node-id`, `t`. `viewport` is
   optional.
5. **Drop** share-only junk like `p=f`.
6. `node-id` may use `:` or `-` between numbers — both work in the URL
   (`1341-135299` ≡ `1341:135299`). When the same node id is also needed for the
   MCP/REST API, use the colon form there.
7. Also set `ORIGINAL_PROTO_URL` to the plain `www.figma.com/proto` version of the
   same link (so the "Figma ↗" button opens the normal player).

Example shape of a correct embed URL (placeholders — never commit a real `t=`
share token to this public repo):

```
https://embed.figma.com/proto/<FILE_KEY>/<FileName>?page-id=<PAGE_ID>&node-id=<NODE_ID>&t=<TOKEN>&scaling=contain&content-scaling=responsive&starting-point-node-id=<START_NODE>&footer=false&viewport-controls=false&device-frame=false&embed-host=figma-prototype-zoom
```

### In-app link editor (built in)

The presenter has a **"Link" button** in the dock (next to the "Figma ↗" button)
that toggles a compact top panel — a single row: a URL input with an inline **×
clear** button, plus a **Load** button:
- The pasted link is **persisted** in `localStorage` (key `figma-prototype-zoom`,
  field `link`) as the raw string the user typed, and auto-loads next time.
- On load and on apply, the raw link is run through **`normalizeFigmaLink()`**, which
  applies the whole recipe above automatically (host → embed, force
  `scaling=contain` + `content-scaling=responsive`, add embed params, drop `p`). So
  the user can paste a messy share link and it is auto-corrected.
- Invalid input (not a URL / not figma.com / no file key) shows an inline error and
  does NOT change the loaded prototype.
- The panel hides after a successful Load or on a click outside it (no Close button).
  Outside-click is caught by a transparent full-screen overlay (`#linkOverlay`,
  shown only while open) so clicks over the cross-origin Figma iframe close it too
  (those clicks never reach the parent document otherwise). The **×** clears the
  input text (no Delete button).
- While the panel is open, the dock **Link** button gets the `.active` class
  (turns blue) as an "open" indicator. NOTE: `getComputedStyle().backgroundColor`
  misreports this in the headless preview — trust a screenshot, not eval, for the
  blue state.
- The status line is `hidden` when empty (no reserved bottom gap); it only appears
  to show an error or the "loaded/normalized" confirmation.
- `DEFAULT_LINK` is `""` (public tool, no baked-in prototype). With no saved/valid
  link the empty state shows: `.no-link` on `<body>` hides the iframe + dock, shows
  the top link panel by default, and reveals `.welcome-center` (centered hint text +
  dashed up-arrow). It is NOT a separate UI — the link panel is reused as-is.
- The same `normalizeFigmaLink()` also produces the plain `www.figma.com/proto`
  URL for the "Figma ↗" button.

If you change the URL rules, change them ONLY inside `normalizeFigmaLink()` — it is
the single source of truth for link building.

### Adding MULTIPLE prototype links later

The presenter currently holds ONE `EMBED_URL`. To support several screens:
- Keep a small array/manifest of `{ label, embedUrl, protoUrl }`, each built with
  the recipe above.
- Add a switcher in the dock (or reuse the scale row). Switching the iframe `src`
  reloads the embed (a brief reload is acceptable on an explicit user action).
- Apply the recipe to EVERY new link — the `scaling=contain` +
  `content-scaling=responsive` rule is per-link and must never be skipped.

## Zoom, fullscreen and rendering behaviors (don't regress these)

- **Zoom = reflow, not bitmap magnify.** Steps are **100 / 125 / 150** (MAX_ZOOM
  `1.5`). At `zoom > 1` the iframe is sized to `stage / zoom` px and then
  `transform: scale(zoom)` (origin `0 0`). This gives Figma a *narrower viewport*
  so it re-lays-out responsively and content stays within the edges. Plain
  `transform: scale()` on a full-size iframe just magnifies the bitmap and pushes
  bottom/right elements off-screen — the user explicitly rejected that.
- **Sharp zoom is NOT achievable here — do not re-attempt it.** The prototype is a
  cross-origin OOPIF; the parent cannot raise its devicePixelRatio or force it to
  re-render larger, and `content-scaling=responsive` *reflows* on resize instead of
  scaling (so a full-size iframe doesn't zoom, it just widens + scrolls). So the
  scale-up softens pixels at high zoom. This is structural, not a bug. Decision
  (researched + adversarially verified): **accept mild softness**, cap zoom at 150%,
  and tell users that real crisp magnification is the browser's own ⌘/Ctrl-+.
  The ONLY way to get truly sharp zoom is `content-scaling=fixed` — but that throws
  away the responsive "real website" behavior and brings back per-page jumps, so it
  was rejected. Don't reopen "sharp + responsive": they are mutually exclusive.
- **The dock has a `Reload` button** (not Reset): it re-assigns the iframe `src`
  with a `reload=<ts>` cache-bust to pull the latest published version (the embed
  link always serves latest; there is no token-free way to *detect* an update).
- **At `zoom == 1` the iframe must be as plain as possible** (no `will-change`,
  the transform cleared). A persistent compositing layer caused black flashes on
  page navigation in windowed mode.
- **Force Figma to re-fit after any size change with a "nudge".** Figma's embed
  leaves the canvas fit to the OLD size (→ black bars) when zooming back down or
  entering fullscreen. Mimic a real window resize: write a size a few px off, then
  the correct size ~70ms later (a single rAF is too fast — Figma coalesces it). On
  `fullscreenchange`, re-fit immediately and again after the macOS fullscreen
  animation settles.
- **Fullscreen black bars during navigation = the `::backdrop`.** The fullscreen
  element's `::backdrop` defaults to BLACK and flashes through while Figma
  re-composites its cross-origin iframe. Fix:
  `:fullscreen::backdrop { background: var(--stage-bg) }` (+ `-webkit-` variant)
  and `:root:fullscreen { background: var(--stage-bg) }`.
- **Dock is lifted** (`bottom: ~42px`) so it clears Figma's own bottom prototype
  controls (page nav / restart). The two UIs must not overlap.
- **Supersampling is incompatible with responsive layout.** Enlarging the iframe
  to "render at higher resolution" shifts the responsive breakpoint and makes the
  design render tiny/centered. Don't reintroduce it. Sharpness relies on the
  display DPR.

## Previewing / verifying — important limitation

**The Figma embed does NOT load inside the Claude preview sandbox.** It is served
over `http://localhost`, and Figma refuses to be framed there (`net::ERR_ABORTED`
on the `embed.figma.com` request). So you can only verify the **container math**
(iframe size, transform, URL params) locally via `preview_eval`. The actual Figma
rendering — black bars, jumps, fullscreen, navigation — **must be checked by the
user in a real browser.** State this honestly rather than claiming a visual fix.

### Manual test checklist (for the human, in a real browser)

1. Pages don't jump bigger/smaller when navigating between menu items.
2. Fullscreen fills the screen, header pinned top, no black bars top/bottom.
3. Fullscreen → clicking menu items: no "small → big" jump on first visit.
4. Zoom steps are 100 / 125 / 150; each enlarges and reflows, nothing spills
   off the right/bottom edge (mild softness at 150% is expected/accepted).
5. Zoom 150→100 returns cleanly with no black bars.
6. The dock doesn't overlap Figma's own bottom controls.
