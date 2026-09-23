# Design: Click it, edit it inline — no mode-switching (App Studio Designer)

## Problem statement

Binoy, this session, after reviewing the shipped click-to-select work and the qoboto clone:

> "On the main UI, I see Layout and Pages as two links. A user goes to layout and change layout
> and goes to page and edit page. Both will take the user to Designer WYSIWYG. However I am
> thinking this model is killing the usability... I need the best user experience - otherwise
> nobody will use apps like this."

And the concrete spec:

> "when user clicks on the content, it overlays a Editor text box on top of it and user directly
> edits the content and leave. And there should be a pages menu on top and when user clicks on it,
> the page switches. When user clicks on pages main menu, the pages should appear as a sidebar and
> user can switch using that as well."

This supersedes part of `app-studio-page-first-design.md` (2026-08-26) — that target correctly
moved Pages to be the default landing and gave each Page its own scoped canvas, but it kept
**four separate body destinations** (Pages list / Page canvas / Layout Design / App Config) and a
**docked, fixed-width side panel** for widget/section editing. This design collapses that into one
continuous canvas with in-place overlay editing, per Binoy's explicit spec above.

## Current state (confirmed by direct read, 2026-08-30)

### The four-way body switch — `AppShell.tsx` (apps/app-studio-designer/src/layout/AppShell.tsx:165-197)

```
selectedAppId === null       → ProjectTreePanel / AppTreePanel (app picker)
showAppConfig                → AppConfigScreen (full-screen, replaces canvas)
designerMode === 'layout'    → CanvasPanel (Section list + docked Details panel)
editingPageID !== null       → PageCanvas (page-scoped preview + docked widget sidebar)
else                         → PagesPanel (full-screen grid of page cards)
```

Four mutually-exclusive full-screen destinations, driven by `designer-ui.store.ts`'s
`designerMode: 'pages' | 'layout'`, `editingPageID: number | null`, `showAppConfig: boolean`.
`DesignerToolbar.tsx` drives the switches via three buttons: **Pages** (`goToPagesLanding()`),
**Layout Design** (`openLayoutDesign()`), **App Config** (`toggleAppConfig()`).

### Two different widget-editing UIs already exist, doing the same job differently

- **`CanvasPanel.tsx`** (Layout Design mode, lines 100-118, 212-258): `LivePreviewPanel` already
  wires `onSelectWidget`/`onSelectSection` to a **docked right panel** (320px fixed,
  `styles.detailsPanel`) showing `WidgetDetailsContent` (wraps `WidgetEditorFields`,
  Configuration/Style tabs) or `SectionDetailsContent` (5 accordions: Actions/Widgets/Region/
  Widget Layout/Styling). This is this session's own shipped work — the foundation, not something
  to redo.
- **`PageCanvas.tsx`** (lines 142-150): `LivePreviewPanel` is mounted here too, but
  `onSelectWidget` is **never passed at all**, and `onSelectSection` is a no-op
  (`() => { /* section selection is a Layout Design concept, not used here */ }`). Widget editing
  here still goes through the **old `EditWidgetModal`** (line 204-208) opened from a **docked
  300px sidebar's** `WidgetChip` list (lines 152-192) — a real, live gap: clicking a widget
  directly in the page-scoped canvas's own live preview today does nothing.
- **`LivePreviewPanel.tsx`** (lines 76-96): the click-delegation itself (`[data-widget]` /
  `[data-section]`, widget-priority-over-section) is shared machinery already used by both hosts —
  correctly built as host-agnostic (`onSelectWidget`/`onSelectSection` are just optional callback
  props). This is the one piece that needs zero rework to become the click surface for the unified
  canvas.

### `WidgetEditorFields.tsx` — the "two-tab" foundation already exists

Configuration / Style tabs (lines 198-213), config tab branches per `widget.widgetType` (form /
content / site-branding / signin / notifications / hil-inbox). The **content** widget type already
has an "Expression Builder" and "HTML Editor" launcher button (lines 289-313) next to a raw
`<textarea>` — closer to Binoy's "inline HTML editor" ask than any other widget type, but still a
**launcher that opens its own overlay**, not literal in-place editing of the rendered text.

### `PagesPanel.tsx` — the sidebar's source material

Already a full CRUD surface (create/edit/reorder/delete/set-default, menu placement fields) — this
is the component that becomes the slide-out drawer's content, not something to rebuild.

### Routing gap confirmed

`useAppIdUrlSync.ts`'s own doc comment: "single-canvas visual designer, not a multi-page app" — no
per-page URL segment exists in the Designer today (unlike `app-player`, which already has
`/{appCode}/page/{slug}`). The unified canvas needs page-in-URL for deep-linking a specific page's
edit view, matching the pattern already proven in `app-player`.

## Part 0 — the target shape

One continuous canvas. No "Pages" vs "Layout Design" vs "App Config" toolbar buttons driving
separate full-screen views for anything that is *content*:

```
┌─────────────────────────────────────────────────────────────────┐
│ DesignerToolbar: [☰ Pages] Home  About  Contact  [+]   …  Save   │  ← top page-switcher (tabs)
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   [live rendered page — the real AppPlayer/AppEngine render,     │
│    studioMode=true, exactly as today's LivePreviewPanel]         │
│                                                                   │
│         ┌───────────────────────┐                                │
│         │ (clicked heading)     │  ← inline overlay, anchored    │
│         │ [Edit Content ▾]      │    to the clicked element,     │
│         │ ┌───────────────────┐ │    not a docked side panel     │
│         │ │ <rich text box>   │ │                                │
│         │ └───────────────────┘ │                                │
│         │ [Advanced ▸]          │  ← collapsed 2nd tab            │
│         └───────────────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
        ↑ clicking "☰ Pages" slides in a drawer over the canvas:
┌──────────────┬──────────────────────────────────────────────────┐
│ PAGES         │ [live canvas, dimmed/inert while drawer is open] │
│ ▸ Home (def.) │                                                  │
│   About       │                                                  │
│   Contact     │                                                  │
│ + New Page    │                                                  │
└──────────────┴──────────────────────────────────────────────────┘
```

Two destinations remain genuinely separate (justified below, not force-fit into the canvas):
**App Config** (non-visual app settings — name/description/icon, not "content on the page") and,
initially, **Layout Design** for site-wide chrome — see Part 3 for why chrome gets a lighter-touch
treatment than a full removal.

## Part 1 — the overlay panel (replaces the docked Details panel)

**Decision: overlay/popover anchored to the clicked element, not a docked side panel — this is
what Binoy explicitly asked for ("overlays a Editor text box on top of it"), and it's what Wix
actually does.** The shipped docked panel (`CanvasPanel.tsx`'s `styles.detailsPanel`, 320px) is
real, working, un-thrown-away logic — it gets *repositioned*, not rebuilt: same
`WidgetDetailsContent`/`SectionDetailsContent`/`WidgetEditorFields` components, wrapped in a new
positioning shell.

- **Anchoring**: on click, read the clicked DOM element's `getBoundingClientRect()` (the
  `[data-widget]`/`[data-section]` element `LivePreviewPanel`'s delegation already resolves) and
  position a floating panel next to it — right side if there's room, left/below otherwise, same
  collision-avoidance any tooltip/popover library provides. Use `@floating-ui/react` (new
  dependency — small, no heavy deps, the de facto standard for this exact problem; do not
  hand-roll viewport-edge math) rather than a bespoke positioning function.
- **Modality**: non-modal — the canvas behind stays interactive (Wix's own panel doesn't block the
  rest of the page). Clicking a *different* widget/section swaps the panel's target and content in
  place (same component, new props) rather than closing/reopening. Clicking empty canvas space
  (outside any `[data-widget]`/`[data-section]` and outside the panel itself) closes it.
- **Narrow-viewport fallback**: below a width threshold (e.g. the Designer app itself is rarely
  used at phone width, but a split-screen/laptop-narrow browser window is realistic), degrade to a
  **docked bottom sheet** instead of a floating popover — same component tree, different CSS
  container, chosen once at render time via a `useMediaQuery`-style width check. This is the one
  concession to "keep a docked mode" — narrow-viewport only, not the default.
- **Two-tab reconciliation**: keep `WidgetEditorFields`'s existing Configuration/Style tabs
  structure as-is (it's real, shipped, and matches Binoy's original "tab 2 = full config" ask
  exactly) — but for the **content** widget type specifically, the overlay's *default open state*
  should show a lightweight rich-text box bound directly to `content` (see Part 2), with
  "Configuration"/"Style" becoming an explicit second step (a small "More options" affordance),
  not the landing tab. Every other widget type keeps Configuration as its landing tab unchanged —
  content is the one type where Binoy's "tab 1 = direct edit" and "click it, edit it" converge into
  the *same* action, so it gets first-class treatment; the rest don't yet have a materially
  different "direct" mode to offer (form's direct-edit-via-Atlas-Forms-inline-builder is a
  separate, already-scoped design — see the Digital Assets/Atlas Forms design doc in this same
  folder for that integration; wire it into this same overlay's landing-tab slot once it lands,
  don't design it twice here).

## Part 2 — inline rich-text overlay for `content` widgets specifically

This is the literal "overlays a Editor text box on top of it" ask.

- Reuse the **HTML Editor** already built for `WidgetEditorFields`'s content tab
  (`htmlEditor/HtmlEditorLauncher.tsx`, Tiptap-based, lazy-loaded) — it is already a rich-text
  editor component; today it's reached via a button that opens ITS OWN modal launcher. Change: for
  the content-widget overlay's landing tab, mount the Tiptap editor **directly inline** in the
  overlay body (bound to `content`/`contentType==='html'`), not behind an extra "HTML Editor"
  button click. The Expression Builder button and the raw-HTML textarea move to the "More options"
  step alongside Configuration, for the power-user path (dynamic/expression-bound content,
  markdown/plain-text format switch) — they don't disappear, they stop being the default.
- **Markdown/plain-text content types**: the rich-text overlay only makes sense for
  `format === 'html'`. For `markdown`/`text`, the overlay's landing tab is a plain
  textarea-in-place (still inline, still not a launcher) — no WYSIWYG rendering claim for formats
  that aren't WYSIWYG by nature.
- **Save behavior**: matches Binoy's "user directly edits the content and leave" — no explicit
  Save button required for the inline content edit specifically; save on overlay-close (click
  outside, Escape, or select a different widget), same `AppWidgetsApiClient`/`WidgetsApiClient`
  calls `WidgetEditorFields.handleSave` already makes today, just triggered by dismissal instead of
  a button. Keep an explicit visible "Saving…"/"Saved" micro-status (reuse `SaveStatus.tsx`'s
  pattern) so silent-save doesn't read as "did my edit actually stick."

## Part 3 — site chrome / "Layout Design"

Binoy's message groups "Layout" and "Pages" as the two links he wants collapsed — but they are not
symmetric concepts. A Page's content and a site's header/footer/nav chrome are different editing
surfaces in every Wix-class product too (Part 0 of the existing page-first spec already documented
this: "Edit Site Header" is Wix's own deliberately-secondary surface, not folded into the page
canvas). Recommendation: **don't eliminate Layout Design as a concept — eliminate it as a
full-screen toolbar destination.**

- Site-chrome sections (header/footer/nav — anything with a `region` other than the primary
  content flow, or just not `isPrimaryContentSection`) are **already visible and already
  clickable** in the unified canvas's live preview (`data-section` delegation covers them today).
  Clicking the header chrome opens the **same overlay** as any section click — Actions/Widgets/
  Region/Widget Layout/Styling accordions (this is `SectionDetailsContent`, already built, just
  now reachable via overlay instead of docked panel).
- What's removed: the **toolbar's separate "Layout Design" button and `designerMode: 'layout'`
  full-screen mode**. What's removed is the *destination*, not the editing capability — Add
  Section (today's `CanvasPanel` header button) becomes a small "+" affordance at the top/bottom
  edge of the live canvas (Wix's own "Add Section" hover-affordance pattern between existing
  sections), and the left Sections-list-as-navigator (`REGION_GROUPS`/`SectionCard` tree) becomes
  the content of a second slide-out drawer — same drawer mechanism as Pages (Part 4), a different
  trigger ("Site Structure" or similar, next to the Pages menu icon), not a new full-screen route.
- Net effect: Layout Design's *capability* survives 100% (nothing in `useSectionActions`,
  `SectionCard`, `moveSection`/`renameSection`/`removeSection` changes) — only its *reachability*
  changes, from a toolbar mode-switch to an in-canvas click plus one drawer for the rarer
  "manage all sections at once" case.

## Part 4 — top page-switcher + Pages drawer

- **Top tab bar**: replaces `DesignerToolbar`'s standalone "Pages" button. Reads the app's
  `menuPages` (same `AppPagesApiClient.list()` data `PagesPanel` already fetches) and renders each
  page's title as a clickable tab, current page highlighted. Clicking a tab calls the same
  `engine.previewNavigate({ pageID })` `LivePreviewPanel`/`useLivePreviewEngine` already use for
  `PageCanvas`'s page-scoped preview (Part 2 of this doc's "Current state" — this mechanism moves
  from PageCanvas's mount-effect to the tab bar's click handler; no new engine capability needed).
  A `☰`/hamburger icon at the tab bar's start is the drawer trigger (Part 0's sketch).
- **Pages drawer**: `PagesPanel.tsx`'s existing component, rendered inside a slide-out panel
  (position: fixed, transform-in from the left, dims/inert-izes the canvas behind via a semi-
  transparent scrim) instead of as a full-screen body view. Its `onOpenPage` callback changes from
  "navigate to a new full-screen PageCanvas" to "close the drawer + tell the tab bar/canvas to
  switch to this page" — same data, different host container. New-page creation still lands you
  directly on the new page's (now-inline) canvas per the existing "create & open" convention
  (`PagesPanel.tsx` line 413).
- **URL**: extend `useAppIdUrlSync.ts` (or a new sibling hook, since that file's own doc comment
  explicitly scopes it to app-level-only today) with a page segment —
  `/{appCode}/page/{pageSlug}` — mirroring `app-player`'s existing scheme exactly (same
  `RouteResolverService`-adjacent convention, not a new one). This closes the "i want slug for app
  AND page" gap in the Designer specifically (the Player side of that request was already built
  earlier this session).

## Part 5 — App Config

Stays a separate, deliberately-reached surface — full-screen or a drawer (either works; full-
screen is simplest since it's genuinely non-visual: name/description/icon/project, not something
you'd ever want to see "in context" behind a live canvas). No change from today's
`AppConfigScreen.tsx` beyond how it's reached (toolbar button, unchanged).

## Component-by-component migration plan

| Component | Disposition |
|---|---|
| `LivePreviewPanel.tsx` | **Unchanged.** Already host-agnostic; becomes the ONE canvas host instead of being duplicated between `CanvasPanel`/`PageCanvas`. |
| `WidgetDetailsContent` / `SectionDetailsContent` (in `CanvasPanel.tsx`) | **Extracted** into their own files (currently inline in `CanvasPanel.tsx`), **rehosted** inside the new overlay/popover shell. Internal logic unchanged. |
| `WidgetEditorFields.tsx` | **Modified**: content-widget landing tab becomes the inline Tiptap editor (Part 2); Configuration/Style structure otherwise unchanged. |
| `useSectionActions.ts` | **Unchanged.** Already shared between list and details surfaces; the drawer's Sections list and the overlay both keep using it. |
| `SectionCard.tsx` / `REGION_GROUPS` tree | **Rehosted** as the "Site Structure" drawer's content (was `CanvasPanel`'s left column). |
| `PagesPanel.tsx` | **Rehosted** as the Pages drawer's content; `onOpenPage` callback changes (see Part 4). Its own create/edit/reorder/delete logic unchanged. |
| `PageCanvas.tsx` | **Retired** — its two jobs (page-scoped `LivePreviewPanel` + widget sidebar) are absorbed by the unified canvas + overlay. Its `findPrimarySectionName` helper moves to wherever the unified canvas resolves "what page am I on" (likely the new page-switcher hook). |
| `CanvasPanel.tsx` | **Retired as a full-screen route**; its Add-Section button becomes the in-canvas hover affordance (Part 3). |
| `AppShell.tsx`'s 4-way switch | **Simplified to 2**: app picker vs. unified canvas (App Config becomes an overlay-on-canvas or a lightweight 3rd case — implementer's call, not load-bearing either way). |
| `DesignerToolbar.tsx` | **Modified**: "Pages"/"Layout Design" buttons removed; top page-switcher tabs + drawer triggers added in their place. "App Config"/Save/Publish/Versions/Preview unchanged. |
| `EditWidgetModal.tsx` | **Unchanged, kept** — still the correct host for any OTHER caller that legitimately wants a modal (none currently outside the retired `PageCanvas`/`AppConfigScreen`/`PagesPanel`'s own widget references — verify no other consumer before deleting any call site). |
| New: overlay positioning shell | **New component**, `@floating-ui/react`-based, wraps `WidgetDetailsContent`/`SectionDetailsContent` with anchor/collision logic + the narrow-viewport docked fallback. |
| New: top page-switcher tab bar | **New component** in `DesignerToolbar`'s package, reading `AppPagesApiClient.list()`. |
| New: slide-out drawer shell | **New generic component** (used by both Pages and Site Structure drawers — one drawer primitive, two content hosts, per this codebase's own "shared primitive, don't duplicate" convention already visible in `useSectionActions`/`Accordion`). |

## State-model changes

- `designer-ui.store.ts`: `designerMode: 'pages' \| 'layout'` and `editingPageID` **removed**;
  replaced by `activePageID: number | null` (which page the unified canvas is showing — always
  non-null once an app with pages is open) and `openDrawer: 'pages' \| 'structure' \| null` (which
  slide-out, if any, is open). `showAppConfig`/`browseAllApps`/`showVersions` unchanged.
- `app-selection.store.ts`: no changes needed — `selectedSectionKey`/widget selection already work
  the same way regardless of host (docked panel vs. overlay is a presentation concern only).
- New (small): overlay's own anchor state — which DOM rect to position against — is transient UI
  state, owned by the new overlay shell component itself (local `useState`/`useLayoutEffect`), not
  promoted to a store; it has no cross-component consumers.

## Phased implementation plan (rough sizing — sequenced by risk, not just size)

1. **Overlay shell + reposition existing panels** (~3-4 days). Build the `@floating-ui/react`-based
   popover shell; move `WidgetDetailsContent`/`SectionDetailsContent` into it, wired to
   `CanvasPanel`'s existing click handlers first (lowest-risk host, already has both
   `onSelectWidget`/`onSelectSection` wired). Ship this alone, verify live, before touching
   routing/toolbar — de-risks the trickiest new piece (positioning/collision) independent of
   everything else.
2. **Inline content overlay** (~2 days). Content-widget landing tab becomes the inline Tiptap
   editor; save-on-dismiss behavior. Depends on step 1's shell existing.
3. **Unify Pages + Layout into one canvas** (~4-5 days). Retire `PageCanvas.tsx` as a route; make
   the overlay shell from step 1 the click target for a `LivePreviewPanel` that's now the single
   canvas (page-scoped, `pageID`-navigated). Wire the top page-switcher tabs. This is the biggest,
   riskiest step — touches `AppShell.tsx`'s core switch and retires a whole component; do it after
   1-2 are already proven live.
4. **Drawers** (~2-3 days). Build the shared drawer primitive; rehost `PagesPanel` and
   `SectionCard`/`REGION_GROUPS` as its two contents; wire the hamburger/Site-Structure triggers.
5. **Site-chrome in-canvas Add-Section affordance** (~1-2 days). Replace `CanvasPanel`'s header
   button with the hover-between-sections "+" pattern.
6. **Page-level URL routing in the Designer** (~1-2 days). Extend/sibling `useAppIdUrlSync.ts` with
   a page-slug segment.

**Total: ~13-18 working days** for one engineer, sequenced so each phase ships something
independently live-verifiable rather than one big-bang cutover — matches this codebase's own
"live-verify before moving on" convention seen throughout `app-studio-page-first-design.md`'s own
Phase breakdown.

## Constraints (matching this codebase's established convention)

- No auto-commits/pushes without explicit ask.
- `ID` uppercase in every new identifier.
- Additive/backward-compatible where feasible — an app with no pages yet still needs a sane
  landing state (mirrors today's `PagesPanel` auto-Home-page bootstrap).
- `pnpm typecheck`/`pnpm build` clean for every touched package.
- Dated `DevelopmentHistoryLog.md` entries in every touched package.
- Live-verify per phase (per the phased plan above) in both `app-studio-designer` and confirm no
  regression in `app-player`'s standalone runtime (unaffected by this design — it's Designer-only,
  but `LivePreviewPanel`'s underlying `AppPlayer`/`AppEngine` render path is shared, so a quick
  smoke check is warranted after step 3 specifically).

## Open questions for Binoy (flag, don't guess)

1. Should the "Site Structure" drawer (Layout Design's replacement) be reachable from every page's
   canvas, or only relevant when actually on a chrome section? Recommendation: always reachable
   (matches Wix's own persistent "Site" vs. "Page" panel switcher) — but confirm before building.
2. Overlay dismiss-to-save for content widgets (Part 2) — confirm this is the wanted behavior vs.
   an explicit Save button even for the inline case; "leave it" in Binoy's own wording suggests
   dismiss-to-save, but worth a one-line confirmation before it becomes the pattern every other
   widget type's landing tab eventually follows too.

## Build Progress

**Phase 1 — Overlay shell + reposition existing panels: DONE, live-verified (2026-08-30).**

What shipped:
- Added `@floating-ui/react` (`^0.27.0`) to `app-studio-designer-components-react`'s
  `package.json` — the only new dependency this phase needed (Tiptap was already present).
- New `src/overlay/OverlayPanel.tsx` — the popover shell. Non-modal (`useDismiss` with
  `outsidePressEvent: 'mousedown'`, `useRole`, `FloatingFocusManager` with `modal={false}`),
  positioned via a **virtual reference element** (`refs.setPositionReference({ getBoundingClientRect
  })`) rather than a real DOM ref — deliberate, since the clicked source element can unmount/
  remount as selection state changes, but the click's rect is still the right anchor regardless.
  `flip`/`shift`/`offset` middleware, plus a `size` middleware capping max-height to the available
  viewport space. Below `NARROW_VIEWPORT_BREAKPOINT` (720px) it renders as a fixed bottom sheet
  instead — same content, different container, chosen once at render via a `resize`-driven hook.
- Extracted `WidgetDetailsContent`/`SectionDetailsContent` out of `CanvasPanel.tsx` into their own
  files under a new `src/details/` folder — internal logic unchanged, now rehosted inside
  `OverlayPanel` instead of the retired docked 320px side panel.
- `CanvasPanel.tsx` rewritten: `anchorRect` state (type `AnchorRect`, exported from
  `OverlayPanel.tsx`) replaces the old `detailsPanel`/`detailsCollapsed` docked-panel state. Two
  anchor sources wired: (1) `LivePreviewPanel`'s `onSelectSection`/`onSelectWidget` now take an
  optional second `DOMRect` arg (the clicked `[data-widget]`/`[data-section]` element's own rect —
  additive, backward-compatible for any caller that ignores it), and (2) a new capture-phase
  `onClickCapture` on the left Sections-list container reads `data-section-anchor`/
  `data-widget-anchor` attributes (added to `SectionCard.tsx`/`WidgetChip.tsx`'s root elements) —
  fires before the bubble-phase `onSelect`/`onEditWidget` handlers, so `anchorRect` is always set
  correctly before selection state updates, regardless of which surface (WYSIWYG canvas or left
  list) the click came from.
- **Real, unplanned mid-Phase-1 discovery**: Task 5 (undo/redo, running in parallel) landed
  `mutateLayoutWithUndo` in `app-selection.store.ts` while this phase was extracting
  `SectionDetailsContent`'s `patchSection` function — the exact function Task 5's own design doc
  says should route through `mutateLayoutWithUndo`. Reconciled: `SectionDetailsContent.tsx`'s
  `patchSection` now calls `mutateLayoutWithUndo` instead of a raw `useAppSelectionStore.setState()`
  + `markDirty()`, so section-style/region/widgetLayout edits stay covered by undo/redo. This is
  exactly the kind of shared-file collision the coordinating session sequenced 2-before-6-before-3
  to avoid at the CanvasPanel/rendering level — Task 5's store-layer overlap wasn't part of that
  sequencing (it's additive/isolated per the wave plan) but still needed a one-function reconciliation
  once both agents touched the same call site in the same session. Flagging this pattern for
  whoever builds Phase 3/6: check `app-selection.store.ts`'s current state before assuming this
  doc's original code snippets are still exact — Task 5 may have moved things again since.
- `pnpm tsc --noEmit` clean for the whole `app-studio-designer-components-react` package.
- **Live-verified** in `app-studio-designer` against the real qoboto app (AppID 1526,
  `http://localhost:6109/qoboto`, Layout Design mode):
  1. Clicking the hero `<h1>` heading directly in the WYSIWYG live preview opens a floating
     "WIDGET DETAILS" popover anchored right next to it — Configuration/Style tabs, real widget
     data (`Widget Name: "Qoboto: Hero Heading"`, `Content: <h1 id="home">Your brand. Your world.
     One link.</h1>`), Save/Cancel, "← Back to section" link, close (×) button. Canvas behind stays
     visible/interactive (non-modal).
  2. Clicking outside the popover dismisses it (`useDismiss` confirmed working).
  3. Clicking "header-brand" in the LEFT Sections list (not the live preview) opens a "SECTION
     DETAILS" popover anchored next to that list item — all 5 accordions present and populated
     (Actions with move-up/down + Delete Section, Widgets showing both real widgets `#624`/`#625`
     with Add Widget, Region, Widget Layout with Direction/Gap/Align/Justify, collapsed Styling).
  4. No console errors, no regressions in the qoboto site's existing rendering.

**Phase 2 — Inline content overlay: DONE, live-verified end-to-end (2026-08-30).**

What shipped:
- Extracted the real Tiptap editor instance (toolbar + `EditorContent`, no `Modal` chrome) out of
  `HtmlEditorLauncher.tsx`'s `HtmlEditorModal` into a new standalone `htmlEditor/
  HtmlEditorSurface.tsx` — a plain controlled component (`value`/`onChange`). `HtmlEditorModal`
  itself now just wraps this surface with local buffered state + its existing Cancel/OK footer —
  behavior unchanged for every existing caller of `HtmlEditorLauncher` (the Expression-Builder-
  adjacent "More options" path, and the modal-hosted `EditWidgetModal` path).
- `WidgetEditorFields.tsx`: new `enableContentLanding?: boolean` prop (default `false`) gates a
  new landing view — when `true` and the widget is `content`-type, the component renders
  `HtmlEditorSurface` (or a plain textarea for markdown/text formats) directly as the FIRST thing
  shown, with a "More options ▾" link to reach the unchanged Configuration/Style tab bar (which
  gained a matching "▴ Simpler view" link back). No Save/Cancel footer in the landing view.
  Deliberately opt-in, defaulting `false`: `EditWidgetModal.tsx` (AppConfigScreen/PagesPanel/
  PageCanvas's shared modal host) does NOT enable it and is therefore byte-for-byte unaffected —
  it has no dismiss-triggered save wiring, so silently defaulting the footer-less landing view on
  there would let a modal Cancel/close discard edits with nothing to commit them.
- `WidgetEditorFields` is now wrapped in `forwardRef`, exposing `{ saveIfDirty(): Promise<void> }`
  — calls the existing `handleSave()` only when `widget.widgetType === 'content' && content !==
  initialContent` (a real dirty-check against the loaded value, not a blind save on every close).
  A safe no-op for every other widget type / every caller that never enables landing mode.
- `WidgetDetailsContent.tsx` is now also `forwardRef`, forwarding straight through to
  `WidgetEditorFields`'s ref and passing `enableContentLanding` — this is the ONLY host that opts
  in (matches Task 2's design doc: this is a Wix-editor-specific behavior, not a global
  `WidgetEditorFields` change).
- `CanvasPanel.tsx` holds the ref and calls `saveIfDirty()` at every point a content widget's
  landing view could be left without its own Save button: `closeOverlay` (dismiss/× the whole
  overlay), `handleSelectSection`/`handleSelectWidget` (swapping to a different section/widget —
  Part 1 of the design doc is explicit this is an in-place swap, not a close, so the dismiss-only
  hook alone would have silently dropped this case), and the "← Back to section" link (the only
  other exit from the landing view besides closing the whole overlay). All fire-and-forget
  (`void ...saveIfDirty()`) — safe because `handleSave`'s closure captures the widget's own data
  independent of whether its component is still mounted afterward.
- `pnpm tsc --noEmit` clean for `app-studio-designer-components-react` AND `app-studio-store-react`
  (the latter touched concurrently by the parallel Task 5 fork — see the Phase 1 reconciliation
  note above).
- **Live-verified** against qoboto's real hero heading widget (`#627`, "Qoboto: Hero Heading"):
  clicking it now shows the real Tiptap toolbar (undo/redo, bold/italic/underline/strike, H1/H2,
  alignment, colors, link/image/table) directly over the rendered "Your brand. Your world. One
  link." text — no Configuration tab, no textarea, "More options ▾" reachable below. Typed a test
  edit, closed the overlay, and confirmed via a direct `GET /api/v1/app-studio/widgets/627` call
  that the edit persisted server-side (`configuration.content` and `lastModifiedOn` both reflected
  it) — the save-on-dismiss path genuinely works end-to-end, not just visually. Reverted the test
  edit back to the original heading afterward via a direct `PUT` (qoboto's real content, not scratch
  data, so left it clean).

Non-blocking issues found and diagnosed during verification (neither is a defect in this phase's
own code):
- A transient `RangeError: Duplicate use of selection JSON ID cell` from `@tiptap_extension-table`
  appeared during live-editing, cascading into repeated Vite HMR reload failures for several
  touched files. Root cause: ProseMirror's `Selection.jsonID` registry is module-global, not per-
  editor-instance, and Vite's HMR re-evaluated the `@tiptap/extension-table` module while an old
  registration was still live. A hard reload (not just HMR) cleared it and normal operation
  resumed — this is a known class of dev-server-only artifact for singleton-registry libraries
  under HMR, not something that would occur in a real build or fresh page load. Flagging in case a
  future phase hits the same symptom while iterating.
- The backend (Consolidated WebApi) returned several genuine SQL execution timeouts
  (`/apps/1526/pages`, `/app-studio/widgets`, `/ai/template/data-template/by-type`) while multiple
  parallel Wave-1 agents (this task, Task 1's build, Task 4's SEO tab) were all hitting it at once
  from the same dev machine — the same pre-existing memory-constrained-dev-box contention this
  session's memory already documents, not a regression from this phase's code. Verification above
  worked around it by querying the backend directly rather than fighting a contended UI.

**Phases 3 and 4 — Unified canvas + drawers: DONE, live-verified end-to-end (2026-08-30).**
Coordinator confirmed reduced backend contention and directed continuing through Phase 6,
resolving the Site Structure open question with the doc's own recommended default
(always-reachable, flagged rather than blocked on). Implemented Phase 4 (drawers) together with
Phase 3 rather than as a separate pass — they turned out to be tightly coupled (the overlay-open
handlers a drawer-hosted click needs to reach live locally inside the same component that owns
`selectedWidget`/`anchorRect`/the ref, so splitting them across two components would have meant
lifting that state to the global store for no real benefit) — logged together here for that
reason, not because Phase 4 was skipped.

What shipped:
- **State model**: `designer-ui.store.ts` rewritten — `designerMode`/`editingPageID` removed,
  replaced by `activePageID: number | null` (which Page the unified canvas shows) and
  `openDrawer: 'pages' | 'structure' | null`. `resetDesignerView()` now just clears these three.
- **Shared pages loader**: new `pages/useAppPages.ts` hook, extracted from `PagesPanel.tsx`'s own
  inline load/bootstrap-Home-page logic (behavior unchanged). `PagesPanel.tsx` refactored to use
  it (dropped its duplicated `useState`/`useCallback`/`useRef` — one `handleMove` optimistic-update
  callsite needed adjusting to reload-after-success instead, since `pages` is no longer a local
  setter it owns).
- **Real race avoided, not just tolerated**: `DesignerToolbar` (page-switcher tabs) and
  `CanvasPanel` (unified canvas) are both always-mounted siblings under `AppShell` — if each called
  `useAppPages` independently, two concurrent instances could both see an empty pages list on a
  fresh app and each fire the hook's own Home-page-bootstrap, creating two Home pages. Fixed by
  having `AppShell.tsx` own ONE `useAppPages` call and pass `pages` down as a prop to both
  (`DesignerToolbar`'s new `pages` prop, `CanvasPanel` reads `activePageID`/pages-driven state via
  the store instead). `PagesPanel`'s own instance (mounted lazily, only once the Pages drawer is
  actually opened) keeps its own hook call since by the time it mounts the top-level fetch has
  already resolved — verified live: only one "Home" page ever existed, no duplicate.
- **`CanvasPanel.tsx` rewritten into the unified canvas**: no more left Sections-list column: full-
  width `LivePreviewPanel` navigated to `activePageID`, same overlay wiring as Phases 1-2 unchanged
  (`handleSelectSection`/`handleSelectWidget`/`closeOverlay`/`widgetEditorRef`). Empty-layout state
  now points at the Site Structure drawer instead of showing its own inline "Add Section" button.
- **New `overlay/Drawer.tsx`**: shared slide-out primitive (scrim + left-anchored panel, click-
  outside or × closes) — same technique `OverlayPanel`'s narrow-viewport bottom sheet already used,
  anchored to the left edge instead of the bottom.
- **New `pages/SiteStructurePanel.tsx`**: the old `REGION_GROUPS`/`SectionCard` tree + Add Section,
  extracted verbatim from `CanvasPanel.tsx`'s retired left column, rehosted as the Site Structure
  drawer's content. Its own capture-phase click handler reads the same `data-section-anchor`/
  `data-widget-anchor` markers Phase 1 established, so selecting something here opens the exact
  same `OverlayPanel` the WYSIWYG canvas does — one anchoring contract, three trigger surfaces now
  (canvas, this drawer, and previously the left list it replaced).
- **`DesignerToolbar.tsx`**: "Pages"/"Layout Design" buttons removed entirely, replaced by a
  center-aligned page-switcher (hamburger → Pages drawer, layers icon → Site Structure drawer,
  then the actual page tabs — only pages with `showInMenu` per the design doc's own
  `AppPagesApiClient.list()` precedent).
- **`AppShell.tsx`**: body switch simplified from 4 branches to 2 (picker / App Config / unified
  canvas) — `PageCanvas` import and its branch removed entirely. New effect auto-selects the
  default (or first) page once `appPages` loads and nothing is active yet.
- **`PageCanvas.tsx` deleted** (retired per the design doc's own migration table — its two jobs
  fully absorbed by the unified canvas + overlay) and its barrel export removed.
- `pnpm tsc --noEmit` clean for `app-studio-designer-components-react` AND the `app-studio-designer`
  app itself (the actual AppShell.tsx consumer, not just the component library).
- **Live-verified** against the real qoboto app in a dedicated browser tab (per the coordinator's
  isolation instruction): toolbar shows the page-switcher (hamburger, layers icon, "Home" tab
  active, teal-highlighted) with the old Pages/Layout Design buttons gone; the full site (header +
  hero) renders directly in one continuous canvas, no mode switch needed; clicking the hero heading
  still opens the inline Tiptap overlay exactly as Phase 2 built it; the Pages drawer opens showing
  exactly one "Home" page (confirming the race fix works, not just compiles); the Site Structure
  drawer opens showing the full nested section tree (header → header-brand → its 3 widgets, hero →
  hero-left → hero-text, etc.); clicking a widget inside that drawer correctly closed the drawer
  AND opened the Widget Details overlay, anchored near the widget's real canvas position.
- One transient backend stall hit mid-verification (`get-by-id`/`pages` requests pending for
  ~15-20s) — retried per the coordinator's explicit guidance rather than proceeding unverified;
  resolved on its own (confirmed via a direct timed `curl` to the same endpoint: 7.2s response,
  slow but alive) and the retry then rendered correctly. Consistent with the same pre-existing
  dev-box memory-pressure pattern from earlier in this session, not a regression.

**Live feedback fixes (Binoy, while reviewing in-progress work, 2026-08-30) — DONE, live-verified.**
Folded into this same pass per the coordinator's direction rather than treated as a separate cycle.

1. **Fullscreen toggle on every modal/drawer/overlay.** Added once to each of the three shared
   dismissable-surface primitives so every current AND future caller gets it for free, not
   per-instance:
   - `ui/Modal.tsx` — every modal in the app (Create App, Add Widget, Edit Widget, Publish, Pages
     create/edit, HTML Editor) already goes through this one component; added local `fullscreen`
     state + a maximize/minimize `IconButton` in the header, toggling between the existing
     `min(90vw,800px)` default and a new `96vw × 92vh` fullscreen style.
   - `overlay/OverlayPanel.tsx` (Widget/Section Details popover, Task 2 Phase 1) — a third render
     branch alongside the existing anchored-popover and narrow-viewport-sheet branches: bypasses
     floating-ui's anchored positioning entirely and renders a centered, near-full-viewport
     scrim+panel. Resets to the anchored popover on the next `anchorRect` change (a fresh
     selection), not a sticky preference.
   - `overlay/Drawer.tsx` (Pages/Site Structure drawers, Phase 4) — same toggle, widens the fixed
     420px column to 96vw.
2. **Site Structure drawer's transparent-looking background — real bug, fixed.** `Drawer.tsx`'s
   `panel` style used `background: var(--color-bg-secondary, #16161e)` — a CSS custom property
   with no guaranteed-opaque resolution in this context (unlike `OverlayPanel`'s popover, which
   apparently resolves the same variable fine in its own stacking context). Replaced with an
   explicit opaque hex (`#16161e`) on both `panel` and the new `panelFullscreen` style — no more
   trusting a variable that might resolve to something less than fully opaque here.
3. **AddWidgetModal "tiny window" — real bug, root-caused, NOT just a sizing tweak.** Investigated
   live: `.modal-overlay`'s computed `z-index` was `50` (from the shared `@bizfirst/common-themes`
   stylesheet) versus `OverlayPanel`'s `1000`, AND — the actual root cause — `AddWidgetModal` is a
   React child rendered inside `OverlayPanel`'s own popover `<div>` (reached via Section Details'
   "Add Widget" button), which has both `overflow: hidden` and a floating-ui `transform` for
   positioning. An ancestor with `transform` becomes the *containing block* for any
   `position: fixed` descendant instead of the viewport — so `.modal-overlay` (fixed) was being
   clipped down to the ~340px popover's own bounds instead of covering the viewport. Confirmed via
   live DOM inspection: `.modal-overlay` computed to `338×545` while its own child `.modal-content`
   computed to the CORRECT `800×441` — a correctly-sized modal rendered into a clipping context,
   not an actually-tiny one. Fixed with a `createPortal(..., document.body)` in `Modal.tsx` itself
   (matching what `OverlayPanel` already does via `FloatingPortal`) — this fixes the underlying
   stacking-context bug for every modal triggered from inside any transformed/clipping ancestor,
   not just this one reported case, and is the durable fix (bumping z-index alone would not have
   fixed the `overflow: hidden` clipping half of the bug).
4. **Live-verified all four fixes** against qoboto in a fresh dedicated tab (closed and reopened
   mid-verification after a stale-HMR-state crash in the previous tab — see below — to get a clean
   baseline): Site Structure drawer now has a solid opaque background; its fullscreen toggle
   correctly widens it; the Widget/Section Details overlay's fullscreen toggle correctly expands
   it to a centered near-full-viewport panel; `AddWidgetModal` opened from inside Section Details
   now renders as a proper full-size (800×441), correctly-stacked modal on top of the overlay
   behind it, complete with its own fullscreen toggle.
5. **Diagnostic note, not a code bug**: mid-verification, a stale/corrupted browser tab (left open
   across several live Vite HMR reloads of `Modal.tsx`/`OverlayPanel.tsx`/`AddWidgetModal.tsx`
   while actively editing) threw `Error: Target container is not a DOM element` from
   `createPortal`, with no error boundary, crashing the whole `AppShell` React tree (symptom: "No
   layout defined yet"/bare "App #1526" despite the network layer showing every request
   succeeding). Root-caused as the same class of stale-HMR-module-instance artifact already seen
   once this session with `@tiptap/extension-table` (Phase 2's Build Progress entry) — closing
   that tab and opening a genuinely fresh one resolved it immediately and confirmed the real code
   is correct. Flagging the pattern again: hot-reloading a component that owns a portal/singleton-
   registry WHILE it's actively mounted in an open tab is a known source of this dev-only symptom
   class in this environment; a hard tab close+reopen (not just a page reload) is the reliable
   recovery, not a sign of a real regression.

**Phase 5 — In-canvas Add Section affordance: DONE, live-verified, scoped down deliberately.**
Not the design doc's fuller "hover-between-every-section" vision — that requires touching
`AppPlayer.tsx`, the shared rendering package `app-player`'s real end-user runtime also depends
on, which is a materially larger blast radius for a small polish item this late in a long
implementation pass. Shipped instead as a single persistent floating "+ Add Section" pill button
at the bottom-center of the canvas (`CanvasPanel.tsx` only, zero changes to `app-handlers-generic`
or any file `app-player` touches) that opens the same Site Structure drawer the toolbar's own
layers icon does — one Add Section entry point, not a second divergent one. Live-verified: the
button renders correctly over the WYSIWYG canvas and opens the drawer on click.

**Phase 6 — Page-level URL routing: DONE, live-verified both directions.**
`useAppIdUrlSync.ts` extended from `/{appCode|appID}` to `/{appCode|appID}[/page/{slug}]`,
mirroring `app-player`'s own `RouteResolverService`-driven scheme exactly. `appPages` is a
PARAMETER (from `AppShell.tsx`'s own single `useAppPages` call), not a fetch owned by this hook —
joining the same "one fetch, shared" contract Phase 3 already established, rather than adding a
fourth concurrent instance. New `pendingPageSlug` state holds a URL-parsed slug that can't resolve
yet (`appPages` depends on `selectedAppId` existing first) until a dedicated effect resolves it
against `appPages` once that list actually arrives; a stale/unmatched slug is dropped silently
(the unified canvas's own default-page effect already covers "no active page" with a sane
fallback). The push direction extends the existing `initialUrlSettled`-gated effect to also read
`activePageID`'s current slug from `appPages`.
- `pnpm tsc --noEmit` clean for `app-studio-designer` (the actual consumer app).
- **Live-verified, both directions**: (1) push — an already-open Designer tab from earlier in this
  session picked up the new logic via Vite HMR and organically updated its own URL from `/qoboto`
  to `/qoboto/page/home` with no user action, confirming the push effect fires correctly on real
  state; (2) adopt — a fresh, cold navigation directly to `http://localhost:6109/qoboto/page/home`
  loaded the full app correctly (toolbar name, "Home" tab active, the complete site rendered) with
  no URL flicker/revert, confirming the mount-time deep-link resolution works end to end.

**Live feedback fix #4 (Binoy, 2026-08-30) — "Add Widget UI must show above Section Details -
z-index": DONE, live-verified.** A residual half of the AddWidgetModal bug fix #3 above didn't
cover: the `createPortal(document.body)` fix solved the CLIPPING problem, but `.modal-overlay`'s
own CSS z-index (`50`, from the shared `@bizfirst/common-themes` stylesheet) was still lower than
`OverlayPanel`'s (`1000`) — portaling siblings to the same container doesn't change their relative
stacking order, only z-index does. Fixed properly rather than as a one-off number: new
`ui/zIndexScale.ts` exports one shared scale (`Z_DRAWER = 900 < Z_OVERLAY_PANEL = 1000 <
Z_MODAL = 1100`), imported by `Drawer.tsx`/`OverlayPanel.tsx`/`Modal.tsx` in place of each file's
own previously-hardcoded number — so the next overlay added to this app inherits correct layering
by construction instead of needing its own z-index guess. `Modal.tsx`'s `.modal-overlay` div gets
an inline `style={{ zIndex: Z_MODAL }}` to override the theme class's `50`, same "override the
un-overridable class property via inline style" pattern already used there for `.modal-content`'s
`max-width`. Live-verified: opening Add Widget from inside Section Details now renders the modal's
own dark scrim correctly ON TOP of (visibly dimming) the Section Details panel behind it, not the
reverse.

**All 6 phases of Task 2 are now complete and live-verified**, plus four rounds of live feedback
folded in mid-pass (fullscreen toggles on Modal/OverlayPanel/Drawer; the Site Structure drawer
background bug; the AddWidgetModal clipping/stacking-context bug and its z-index-scale follow-up).
Reporting back to the coordinating session.

## Post-completion fix (2026-08-30) — permanent-"Loading…" bug in the unified canvas

**Symptom (found live, after the "all 6 phases complete" report above):** a completely fresh
browser tab opened to `/{appCode}/page/{slug}` for a real app (qoboto) — with every underlying data
call (`by-code`, `get-by-id`, `pages`, `widgets`) confirmed resolved 200 — still showed the canvas
permanently stuck on "Loading…" with the bottom status bar reading "No engine active". No console
error, no `loadError` state ever populated. Not a code regression in Phase 3/6 specifically, and
not (only) the well-known transient Consolidated WebApi memory-pressure stall this dev environment
already has documented elsewhere in this session — see root cause below.

**Root cause, confirmed by reading the code (not guessed):** `StoreBackedAppDataLoader.loadApp()`
(`packages/app-studio-designer-components-react/src/preview/StoreBackedAppDataLoader.ts`) awaits
`Promise.all([WidgetsApiClient.list(), AppPagesApiClient.list(appID)])` as part of the live-preview
engine's load sequence (`useLivePreviewEngine.ts`). Every one of these API calls goes through
`fetchJson()` (`packages/app-studio-api-client-js/src/http.client.ts`), which wraps a bare
`fetch()` call with **no client-side timeout whatsoever** — no `AbortSignal`, nothing. When the
backend stalls on any one of these requests (confirmed via the browser's own network panel: two of
three concurrent `GET /api/v1/app-studio/widgets` calls sat at `pending` for 10+ seconds while a
third, identical, concurrent call to the same endpoint succeeded — a real, observed asymmetry, not
a hypothesis), the `fetch()` promise hangs **forever**. `Promise.all` never settles, `loadApp()`
never resolves or rejects, `setLoadGeneration`/`setLoadError` never fire, and the engine's
`app.status` stays `'loading'` permanently with zero way for the user (or any error boundary) to
tell the difference between "still working" and "will never finish."

**Fix:** added a 30-second `AbortSignal.timeout(30_000)` to every `fetchJson()` call
(`http.client.ts`) — the same duration/rationale `@passport/api-client`'s `HttpClient` already
uses elsewhere in this codebase, so this isn't a new convention. A stalled backend request now
fails visibly after 30s (`loadError` populates, `LivePreviewPanel` shows "Preview failed to load:
…") instead of hanging the UI forever with no error. This does **not** fix the backend stalling in
the first place (that's the separate, already-known Consolidated WebApi memory-pressure issue,
worse under this session's own heavy concurrent-agent load) — it fixes the specific bug reported:
a permanently stuck, unrecoverable, silent "Loading…" with no way for a user to know something
went wrong. This is a systemic fix (every app-studio-designer API call now has a timeout), not
scoped narrowly to the live-preview path, since the same no-timeout gap affects every other caller
of `fetchJson` identically.

Also fixed, same pass: `app-selection.store.ts`'s `UndoSelection.widgetId` renamed to `widgetID`
(CLAUDE.md: "ID" uppercase everywhere) — 3 call sites updated, confirmed local-only (no external
references), `tsc --noEmit` clean.

**Verification status — disclosed honestly, not glossed over:** `app-studio-api-client-js`,
`app-studio-store-react`, `app-studio-designer-components-react`, and `app-studio-designer` all
typecheck clean after both changes. **Could not complete a genuinely fresh-tab live
re-verification with real data**: both browser tabs that had a real authenticated session lost it
mid-investigation (real session expiry, unrelated to this fix), redirecting to the actual Passport
login form with browser-autofilled real credentials — which this agent will not interact with
under any circumstance (hard rule against touching credential fields, no exception for autofill).
The sanctioned dev-only fake-auth bypass (`@passport/fake-auth`, used successfully elsewhere this
session) was tried but only bypasses the frontend login gate — it does not grant a real bearer
token the backend will accept for actual data (`getByCode('qoboto')` correctly 401s against a fake
token, confirmed live). The timeout fix itself is verified correct by code inspection and a clean
typecheck across every consuming package, but a real-data, real-session, fresh-tab confirmation
that qoboto's layout now renders is **NOT done** — flagging this precisely rather than claiming a
verification that didn't happen. Recommend: the coordinating session (or Binoy) re-test with a real
session once one is available; if the "Loading…" state still doesn't clear even after 30s (i.e.
`loadError` doesn't populate either), that would point to a second, still-undiagnosed issue beyond
the no-timeout gap fixed here.

No git commits made.
