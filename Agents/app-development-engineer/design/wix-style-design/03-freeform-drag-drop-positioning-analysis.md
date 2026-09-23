# Free-Form Drag-and-Drop Widget Positioning — Analysis & Effort Estimate

Status: **Analysis only — no code written.** Grounded in direct reads of the current App Studio
layout/style/render code (2026-08-30).

## 1. What exists today (confirmed by reading the code, not assumed)

**Layout model is 100% flow-based, at every level:**

- `AppWidgetRecord.displayOrder: number` (`app-handlers-core/src/types/WidgetRecord.ts`) — a pure
  ordinal position within a section. No `x`/`y`/`top`/`left`/`width`/`height` field exists anywhere
  on the widget placement record.
- `AppSection.widgetLayout` (`AppSection.ts`) — `{ direction: 'row'|'column', wrap, gap, align,
  justify }`. This is a flexbox config, not a canvas. It arranges ALL of a section's widgets with
  one shared direction/gap — there is no per-widget override.
- `AppSection.region` — assigns a section to a named CSS-Grid area (`header`/`left`/`right`/`main`/
  `footer`). Also not free-position — it's a coarse, section-level slot system.
- `AppPlayer.tsx`'s renderer (`AppSectionRenderer`/`WidgetSlot`) confirms this in the DOM: every
  wrapper is `display: 'flex'`, every section/widget is a normal flow child. The only
  `position: 'relative'` in the whole renderer is on the **studio-mode-only** dashed-border overlay
  wrapper (`AppSectionRenderer`, line ~252) — real, but currently gated to studio mode and used only
  as an anchor for an overlay label, not for real content positioning.
- `moveAppWidget`/`reorderWidgetsInSection` (`app-selection.store.ts`) both work by swapping/
  rewriting `displayOrder` and PUTting each changed widget individually (no bulk-reorder endpoint
  exists). This is ordinal reordering, not coordinate placement, and would be meaningless inside a
  canvas-mode section (see §4, "blockers").

**The style schema is half-ready, which matters a lot for effort estimation:**

- `StyleProperties` (`atlas-forms/packages/designer-components-react/.../StyleBuilderPanel/types.ts`)
  — the shared structured style editor App Studio's Style Builder already uses for
  `sectionContainer`/`widgetContainer`/etc — **already has** `position: 'static'|'relative'|
  'absolute'|'sticky'|'fixed'` and `zIndex: number`, plus `width`/`height` as strings.
- It does **not** have `top`/`left`/`right`/`bottom` offset fields. So the type is roughly 60% of
  the way to modeling absolute placement, but the actual coordinate fields are missing. Adding them
  is a small, additive, low-risk change to one shared type (same shape as the existing
  `BoxModelEditor` used for padding/margin/border — a `PositionEditor` sibling is a natural fit, not
  a new subsystem).
- Because `useStyleSlot` already casts `StyleProperties` straight to `React.CSSProperties` with zero
  transformation, adding `top`/`left`/`right`/`bottom`/`width`/`height`/`zIndex` values would flow
  through to real inline styles **with no renderer change** — the render-side plumbing already
  exists for arbitrary CSS properties on `widgetContainer`. This is the single biggest reason this
  feature is cheaper than it looks: **no new database columns or DTO fields are needed.** Position
  data can ride inside the existing `AppWidget.styleConfiguration` JSON column exactly the way
  padding/color/border already do.

**A partial breakpoint concept already exists and is directly relevant to Task 6 (flagging, not
solving here):** `AppSection.responsive?: { tablet?: Record<string,string>; mobile?:
Record<string,string> }` and `hiddenOn?: AppBreakpoint[]` are typed and present, but I did not
verify they're actually consumed at render time in `AppPlayer.tsx` — that verification belongs to
the Task 6 (responsive breakpoint editing) design, not here. What matters for **this** doc: any
coordinate schema this feature introduces should be shaped as `{ desktop: {...}, tablet?: {...},
mobile?: {...} }` from day one, even if only `desktop` is implemented first, specifically so Task 6
doesn't force a breaking migration of Task 3's data later. Recommend the two designs sync on this
one schema shape before either is built.

**Precedent search — what canvas/drag tooling already exists in this codebase:**

- `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities` are declared in the top-level
  `BizFirstAiStudio/pnpm-workspace.yaml` catalog, but grep across `src/` shows almost no real usage
  — `WidgetChip.tsx` and `flow-observer-panel/TabBar.tsx` are the only hits, and the Designer's
  existing section/widget drag-reorder (`SectionCard.tsx`'s `draggedId`/`dropTargetId`/`handleDrop`)
  is native HTML5 drag-and-drop (`draggable` attribute + `onDragOver`/`onDrop`), not dnd-kit's API.
  So dnd-kit is available (already in the dependency catalog, zero install cost) but effectively
  unproven in this codebase — budget time for a first real integration, not just "wire up an
  existing pattern."
- **The one mature, production-proven absolute-position canvas in this codebase is
  `flow-studio-designer`'s workflow node editor, built on `reactflow@^11.10.0`** (confirmed in its
  `package.json`) — real pan/zoom, real drag, 40+ node types, already battle-tested. It is the
  strongest existing precedent for "users drag things around a 2D canvas" in this repo. However its
  shape (a node-and-edge graph) doesn't match "place a content widget on a page" — reactflow brings
  a lot of graph-specific machinery (edges, handles, minimap) that's irrelevant here and would be
  wrong-shaped overhead. **Recommendation: study reactflow's interaction/perf patterns (pointer
  capture, transform-based drag instead of top/left mutation during drag, viewport culling) but
  build on dnd-kit's raw primitives (`useDraggable`, `DragOverlay`), not reactflow itself.**

## 2. Recommended strategy: additive "Canvas section" mode, not a layout replacement

Do **not** replace the flow layout engine. Two reasons, one architectural and one product:

- **Architectural**: every additive field this session's own architecture has introduced
  (`region`, `widgetLayout`, `responsive`, `hiddenOn`) follows the same rule — omitted means
  "render exactly as before." That's why the qoboto clone (built entirely on flow sections) still
  renders correctly after this session's nested-section and region-grid work landed. Free
  positioning should follow the same rule: a new optional `AppSection.layoutMode?: 'flow' |
  'canvas'` (default `'flow'`, i.e. unset = today's behavior, byte-identical for every existing app).
- **Product**: Wix itself has been moving *away* from pure free-position as the default. Its
  original Editor (2013-era) was 100% absolute-position and became notorious for unusable mobile
  layouts and accessibility problems, because free position has no inherent responsive behavior —
  every element needs hand-tuned per-breakpoint coordinates or it breaks. Wix's newer "Editor X" /
  Studio product deliberately reintroduced flex/grid-based "smart" layout as the recommended default,
  with free positioning kept as an option for specific marketing/hero-style sections. That's a
  strong signal to copy the *option*, not make it the primary model: keep flow sections as the
  default and recommended path (matches this codebase's existing direction), and offer `layoutMode:
  'canvas'` as an opt-in per-section escape hatch for hero banners and similar marketing layouts —
  exactly the qoboto hero section is the realistic first user of this, not an app's whole page.

## 3. What building this actually requires

1. **Type/schema**: add `top?/left?/right?/bottom?: string` to `StyleProperties`
   (`atlas-forms/.../StyleBuilderPanel/types.ts`) — additive, shared, benefits other features too
   (e.g. sticky headers). Add `AppSection.layoutMode?: 'flow' | 'canvas'`. No `WidgetRecord`/
   `AppWidgetRecord` field changes, no backend DTO/DB changes — position rides in the existing
   `styleConfiguration.widgetContainer.style` JSON.
2. **Style Builder UI**: a `PositionEditor` panel (sibling to the existing `BoxModelEditor`) exposing
   top/left/right/bottom/width/height/zIndex — only shown/relevant when the widget's parent section
   is in canvas mode.
3. **AppPlayer render-mode branch**: `AppSectionRenderer` gets a canvas branch — section wrapper is
   unconditionally `position: relative` (today it only is in studio mode, for the border overlay);
   each `WidgetSlot` inside applies `position: absolute` + the four offsets + width/height/zIndex
   from `styleConfiguration.widgetContainer.style` instead of flowing through `widgetLayout`.
4. **Designer drag interaction**: dnd-kit `useDraggable` (or raw pointer events, given how thin the
   real requirement is) driving a live-preview overlay directly on the WYSIWYG canvas (reusing this
   session's `LivePreviewPanel` click-to-select surface, which already emits `data-widget`
   attributes to hook into) — drag updates a local x/y during the gesture, commits via a new
   `updateWidgetPosition(appWidgetID, {top,left,width,height,zIndex})` store action on drop.
   Wix-parity expectations that materially add scope here: snap-to-grid/alignment guides against
   sibling widgets, resize handles (not just move), keyboard arrow-key nudging, multi-select drag.
5. **Persistence**: `updateWidgetPosition` merges into the widget's existing
   `styleConfiguration.widgetContainer.style` and PUTs through `AppWidgetsApiClient.update` — the
   exact same call shape `moveAppWidget`/`reorderWidgetsInSection` already use, just a different
   payload. No new endpoint needed.
6. **Mode-switch UX**: a section-level toggle (in the new Section Details panel this session
   shipped is the natural home) between Flow/Canvas, with an explicit warning when switching
   Canvas→Flow that absolute coordinates will be discarded (there's no sane auto-conversion back to
   ordinal `displayOrder`).

## 4. Effort estimate (phased, developer-days)

| Phase | Scope | Est. |
|---|---|---|
| 0 | `StyleProperties` position fields + `PositionEditor` UI | 2–3 days |
| 1 | `AppSection.layoutMode` + `AppSectionRenderer` canvas branch in AppPlayer | 3–4 days |
| 2 | Drag interaction (dnd-kit integration, move + resize handles, snap/align guides, keyboard nudge) | 8–12 days |
| 3 | Persistence (`updateWidgetPosition` action, optimistic update, PUT wiring) | 2–3 days |
| 4 | Flow↔Canvas mode-switch UX + data-loss warning | 2–3 days |
| 5 | QA: interaction with nested-section-tree recursion, region-grid coexistence, existing flow sections unaffected | 3–5 days |
| **Total** | **Single-breakpoint (desktop-only) canvas mode** | **~20–27 dev-days (4–6 weeks)** |

This estimate is for **one** coordinate set (no responsive breakpoints yet). Per-breakpoint
coordinates (true Wix parity — independent desktop/tablet/mobile placement) is Task 6's scope and
would add meaningfully more on top of this; it is not included above.

## 5. Blockers / risks

- **Nested sections** (this session's `sectionTree.ts` work): a canvas-mode section with nested
  `appSections` children needs its own coordinate space nested inside a positioned parent — the
  renderer's recursion and the Designer's click-to-select hit-testing (currently relies on flow
  layout's natural DOM stacking order) both need to become z-index-aware for overlapping absolutely
  positioned widgets. Not a blocker, but real added complexity beyond a flat section.
- **Region-grid coexistence**: a `region`-bearing section that's also `layoutMode: 'canvas'` is a
  legitimate combination (e.g. a canvas-mode hero placed in the `main` region) but needs explicit
  test coverage — the two features were designed independently.
- **Responsive breakpoints (Task 6 dependency)**: strongly recommend the coordinate schema
  (`{desktop, tablet?, mobile?}`) be agreed between this design and Task 6's before Phase 0 ships,
  to avoid a breaking data migration later.
- **`moveAppWidget`/`reorderWidgetsInSection` become meaningless inside a canvas section** — the
  Section Details "Widgets" accordion's Up/Down buttons need an explicit decision for canvas mode
  (most likely: repurpose as z-index reorder, not `displayOrder`).
- **Perf**: many absolutely-positioned, independently draggable widgets need careful pointer-event
  and resize-observer handling to stay smooth — reactflow's proven patterns (transform-based drag,
  not live top/left mutation) are worth studying even though reactflow itself isn't the right base.

## Build Progress (2026-08-30)

**Status: real v1 shipped — move + persist + mode toggle + z-index reorder. Resize handles,
snap/align guides, and keyboard nudge explicitly NOT done (stretch goals per the original scope,
correctly deferred, not silently dropped). No live-browser verification performed this pass, per
Binoy's explicit "build now, verify everything together at the end" direction — `pnpm tsc --noEmit`
clean across every touched package is this pass's bar.**

**§3.1 Type/schema** — done exactly as this doc's own §3 recommended:
- `StyleProperties` (`atlas-forms/packages/designer-components-react/.../StyleBuilderPanel/types.ts`)
  gained `top?/left?/right?/bottom?: string`.
- `AppSection.layoutMode?: 'flow' | 'canvas'` added (`app-handlers-core/src/types/AppSection.ts`).
- **No schema changes needed for responsive positioning** — confirmed live that Task 6's
  `ResponsiveStyleValue<T>` already wraps arbitrary `StyleProperties`, so adding the 4 offset
  fields there means position data gets `{base,tablet?,mobile?}` support for free. This doc's own
  §5 "Blockers" flagged syncing the coordinate schema with Task 6 before either shipped — turned
  out to need zero extra work, not a new parallel schema, since Task 6 had already generalized the
  right way.

**§3.2 Style Builder UI** — `style/PositionEditor.tsx` (new), wired into `StyleSlotEditor.tsx` via
a `showPositionEditor` prop, shown only for `widgetContainer` when the widget's parent section is
canvas mode (`WidgetEditorFields.tsx` determines this via `findSectionEntry`). Reuses Task 6's
device-switcher machinery unchanged.

**§3.3 AppPlayer render-mode branch** — done in `AppPlayer.tsx`'s `AppSectionRenderer`/`WidgetSlot`,
exactly as designed: section wrapper gets `position: relative` unconditionally (every runtime, not
just studio mode) when `layoutMode==='canvas'`; each `WidgetSlot` gets `position: absolute` via a
new `canvasMode` prop. The actual offsets needed ZERO new render code — they ride the existing
`StyledSlot` style-merge mechanism for free.

**§3.4 Designer drag interaction** — shipped, but NOT on `@dnd-kit/core` as this doc originally
suggested. Investigated first (added the dependency, then reverted it once the reason became
concrete): the real `[data-widget]` DOM nodes render deep inside `AppPlayer`, a package this
Designer-only panel doesn't own and the real end-user `app-player` runtime also depends on — dnd-kit's
`useDraggable` needs a React-owned ref on the exact draggable node, which would mean threading
Designer-only drag state into shared production rendering code. Built on native pointer events
instead (`LivePreviewPanel.tsx`): `pointerdown` on a widget whose computed `position === 'absolute'`
starts a drag (that computed-style check is the whole "is this canvas mode" signal — no extra prop
threading needed), live feedback mutates the DOM directly (no per-pixel React re-render), drop fires
a new `onWidgetPositionChange` prop wired to the new `updateWidgetPosition` store action. A 4px
movement threshold disambiguates drag from click; verified by reading the existing click-to-select
handler's logic that it's completely unaffected, not assumed.

**§3.5 Persistence** — `updateWidgetPosition`/`reorderCanvasWidgetZIndex` added to
`app-selection.store.ts`, same `AppWidgetsApiClient.update` call shape every other widget mutation
uses. Both correctly clear `undoStack`/`redoStack` on invocation (Category B, per this session's
own established undo/redo-safety convention — verified this was in fact required, as anticipated).

**§3.6 Mode-switch UX** — "Layout Mode" accordion in `SectionDetailsContent.tsx`, Flow/Canvas toggle
with a `window.confirm` warning on Canvas→Flow (offsets aren't cleared, just stop being read).

**Blockers re-checked, resolved**:
- Nested sections: confirmed correct by construction — `layoutMode` is per-section, not inherited,
  so recursion needed no special-casing; each level governs only its own direct widgets.
- Region-grid coexistence: unaffected by construction — `region` (parent placement) and
  `layoutMode` (own children's layout) are orthogonal, canvas mode's code never touches region
  handling.
- `moveAppWidget`/`reorderWidgetsInSection`'s Up/Down in canvas mode: implemented as z-index reorder
  (`reorderCanvasWidgetZIndex`), per this doc's own recommendation.
- Perf (reactflow-style transform-based drag): NOT done — this v1 mutates `style.left`/`style.top`
  directly during drag, not a CSS `transform`. Real, honest gap for a future pass if drag performance
  with many widgets ever becomes visibly janky; not verified either way since no live testing
  happened this pass.

**Verification**: `pnpm tsc --noEmit` clean across all 5 touched packages (atlas-forms
designer-components-react; app-studio app-handlers-core, app-handlers-generic, app-studio-store-react,
app-studio-designer-components-react). No git commits.
