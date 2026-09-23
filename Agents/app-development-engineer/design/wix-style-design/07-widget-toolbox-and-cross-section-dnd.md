# Widget Toolbox + Cross-Section Drag-and-Drop — Design

Status: Design + build combined (Binoy's instruction for this task: "Spec it out as a design doc
and complete the coding for this"). Companion to the other `wix-style-design` docs — this one is a
new post-batch feature request, not one of the original nine tasks; it builds on Task 2's unified
canvas/overlay/drawer shell and Task 3's native-pointer-drag precedent.

## 1. What the user asked for

> "is widget be dragged from one section to another? also a widget can be dragged from a toolbox?
> In the toolbox, i like to see new widgets and existing widgets as toolbox items?"

Two capabilities, neither of which exists today (confirmed by reading the real code):

- **(A) A Toolbox panel** — drag a "new widget type" or an "existing widget" out of a palette and
  drop it onto a section to place it there, instead of the current click-through-a-modal-only flow.
- **(B) Cross-section widget dragging** — drag an already-placed widget chip out of the section it's
  in and drop it on a *different* section, moving it there. Today's drag-and-drop
  (`WidgetChip.tsx`/`useSectionActions.ts`) only reorders within one section — `reorderWidgetsInSection`'s
  own signature takes a single `sectionName`, and each section's drag state
  (`draggedID`/`dropTargetID`) is local `useState` inside `useSectionActions`, scoped per
  `SectionCard`/`SectionDetailsContent` instance — a chip dragged out of Section A's own hook
  instance is invisible to Section B's separate hook instance. That's the concrete reason cross-
  section dragging is "currently impossible," not just unbuilt.

## 2. Current state (verified against source, 2026-08-30)

- **Same-section reorder**: `WidgetChip.tsx` uses native HTML5 drag-and-drop (`draggable`,
  `onDragStart`/`onDragOver`/`onDrop`) — explicitly not `@dnd-kit`/`react-dnd`, matching this
  codebase's own established precedent (see `03-freeform-drag-drop-positioning-analysis.md` §1: "the
  Designer's existing section/widget drag-reorder ... is native HTML5 drag-and-drop ... not dnd-kit's
  API"). `useSectionActions.ts` owns the drag state and calls `reorderWidgetsInSection(sectionName,
  reorderedIDs)`. Two call sites share this hook: `canvas/SectionCard.tsx` (rendered inside the "Site
  Structure" drawer, `pages/SiteStructurePanel.tsx` — Task 2, Phase 4 relocated it there from
  `CanvasPanel.tsx`'s old left column) and `details/SectionDetailsContent.tsx` (rendered inside the
  `OverlayPanel` popover Task 2, Phase 1 introduced).
- **Free-form canvas positioning (Task 3)**: `LivePreviewPanel.tsx` drags real rendered `[data-widget]`
  DOM nodes via native pointer events (`pointerdown`/`pointermove`/`pointerup`), deliberately not
  `@dnd-kit` — that library was tried and reverted specifically because the actual draggable DOM
  nodes render deep inside `@app-studio/generic`'s `AppPlayer`, a package shared with the real
  end-user `app-player` runtime, which has no business depending on a Designer-only drag library.
  This only repositions a widget *within* one canvas-mode section — it is not a section-to-section
  move and this design doesn't change that.
- **Add-widget-via-modal**: `modals/AddWidgetModal.tsx` — a two-tab modal (`'new'`/`'existing'`).
  "New" builds a type-specific `configuration` JSON via `buildNewWidgetConfiguration` (a local,
  unexported function) and calls `WidgetsApiClient.create` then `AppWidgetsApiClient.create`.
  "Existing" lists every `WidgetDto` (`WidgetsApiClient.list()`) and calls `AppWidgetsApiClient.create`
  directly. Both hardcode `displayOrder: 0` and refresh via `refreshAppWidgets()`. `NOT_YET_AVAILABLE`
  (currently `workflow-template-category`, `chat-panel`) are shown but disabled, with an honest
  "not available to create yet" message rather than a silently-broken widget.
- **`AppPlayer`'s DOM already carries `data-section={sectionName}` and `data-widget={appWidgetID}`**
  on every section/widget wrapper (confirmed via `LivePreviewPanel.tsx`'s own click-delegation and
  Task 3's pointer-drag code) — this is what lets both existing Designer-only interactions avoid ever
  touching the shared `AppPlayer` package. This design reuses the exact same trick for a third
  interaction (§5).
- **`app-selection.store.ts` was being concurrently edited by two other efforts today** (per the
  session status): a `structuralGeneration` counter/light-save path, and an atomicity fix to
  `reorderCanvasWidgetZIndex`. Both landed before this design's own store edits — no collision;
  verified the file's current shape immediately before editing it, not from an earlier read.

## 3. The four drag interactions — and how a real user tells them apart

Adding this feature makes FOUR distinct drag gestures exist in the same UI. Wix itself keeps these
visually distinguishable mostly through *where the cursor starts* and *what element moves under it*
— not through a mode toggle — so this design follows the same principle instead of adding new UI
chrome (a legend, a mode switch) that Wix itself doesn't need either:

| # | Drag starts on… | Drag ends on… | What happens | Where it's available |
|---|---|---|---|---|
| 1 | A widget chip's drag handle (⠿ icon), within a section's own widget list | Another chip **in the same section** | Reorders within that section (unchanged v1 behavior) | Site Structure drawer, Section Details popover |
| 2 | A widget chip's drag handle | A chip or empty space **in a different section's** widget list | Moves the widget to that section (new, §6) | Site Structure drawer, Section Details popover, and cross-surface between them |
| 3 | A Toolbox item (new-type or existing-widget row) | Any section's widget list, OR a section in the live canvas | Places that widget into the target section (new, §5) | Toolbox drawer → any of the three drop surfaces above, or the WYSIWYG canvas |
| 4 | A widget already rendered in the **live WYSIWYG canvas**, inside a `layoutMode: 'canvas'` section | Anywhere within that same section (free position) | Repositions it via `top`/`left` (Task 3, unchanged) | Live canvas only, canvas-mode sections only |

The four never collide in practice because their *start* elements are disjoint: #1/#2 start on a
`WidgetChip`'s own visible drag-handle icon (only rendered in the two list surfaces); #3 starts on a
Toolbox row (only rendered in the Toolbox drawer); #4 starts on a real rendered widget in the canvas
(`getComputedStyle(...).position === 'absolute'`, Task 3's own existing signal, gated to canvas-mode
sections). A user never has to consciously pick a "mode" — they just drag the thing they're looking
at, and where they started the drag is what determines the behavior, exactly like #1 already works
today. The one new judgment call this design adds for the user is #1 vs. #2, and that's resolved
implicitly by *where they drop*, not something they have to decide up front — dropping in the same
section reorders, dropping in a different one moves it, both from the exact same drag gesture.

**Visual affordances that make this legible, not just technically true:**
- Every `WidgetChip` already shows a small "drag to reorder" move-icon handle (`FiMove`) — unchanged,
  now honestly describes both #1 and #2 (its tooltip still says "Drag to reorder," which un-changed
  is slightly incomplete now that #2 exists too — updated to "Drag to reorder or move to another
  section," see §7).
- A section's widget-list container gets a highlighted drop-active border (`containerDropActive`,
  §6) whenever *anything* draggable is currently over it — the same visual regardless of whether
  it's #2 or #3, since from the target section's point of view both are simply "something is about
  to be added to me," which is the only fact that section-level UI needs to communicate.
- The Toolbox panel's own hint text ("Drag an item onto any section to place it there") sets
  expectation before the user starts dragging at all — Wix's own Add Elements panel carries the
  same kind of one-line hint.

## 4. Drag mechanics: native HTML5 drag-and-drop, not `@dnd-kit`

Decision: every new drag interaction in this design uses native HTML5 `draggable`/`dataTransfer`,
matching interactions #1 and #4's own established precedent, and explicitly **not** `@dnd-kit`
(declared in the workspace catalog, essentially unused — see `03-...md` §1's own precedent search).
Same reasoning as both prior docs already independently concluded:

- `@dnd-kit`'s primitives (`useDraggable`/`useDroppable`) need a React-owned ref on the exact
  draggable DOM node. For interaction #3 (Toolbox → live canvas), the *drop target* is a real
  `AppPlayer`-rendered section wrapper — a package shared with the production `app-player` runtime
  that has no business knowing about Designer drag state, exactly Task 3's own reasoning for
  rejecting `@dnd-kit` there. Native `dragover`/`drop` events bubble and can be handled via event
  delegation on `[data-section]` (already emitted, zero `AppPlayer` changes needed) instead.
  Interactions #1/#2 (chip ↔ chip) don't even touch `AppPlayer` at all — same native-DnD approach for
  consistency, and because introducing a second drag library for only half the surfaces would be a
  worse inconsistency than reusing one library everywhere it already works.
- Interaction #2 (cross-section) needs a payload that survives being read by a *different React
  component instance's* drop handler than the one that started the drag (Section A's `WidgetChip`
  starts it, Section B's container ends it) — this is exactly what the native `DataTransfer` object
  is *for*: it's carried by the browser across the whole gesture regardless of which component
  handles which event, without any shared React/Zustand state needed to bridge two otherwise-
  independent `useSectionActions` hook instances.

No new dependency, no `@dnd-kit` import added anywhere in this pass.

## 5. New shared drag-payload contract (`canvas/dragPayload.ts`, new file)

Three custom MIME-shaped `DataTransfer` types disambiguate what's being dragged, checked at `drop`
time (never at `dragover` — Firefox restricts real payload reads to the `drop` event; only
type-presence is safe to query earlier, and this design never needs to):

```ts
'application/x-app-studio-widget-chip'      -> { appWidgetID: number; sourceSectionName: string }
'application/x-app-studio-toolbox-new'      -> { widgetType: WidgetType }
'application/x-app-studio-toolbox-existing' -> { widgetID: number }
```

`decodeDragPayload(dataTransfer): DecodedDragPayload` returns a discriminated union (`{ kind, payload
} | null`) — every new drop handler in this design calls this one function rather than re-parsing
`DataTransfer` itself, so the three kinds are decoded identically everywhere. A `text/plain` mirror
of the same JSON is also set (belt-and-suspenders — some browsers require at least one `setData`
call of a "standard" type for a drag to be legal to start at all) but is never the thing a drop
handler reads for real logic.

## 6. Cross-section widget move (§3, interaction #2)

### 6.1 New store action

`app-selection.store.ts` gains `moveAppWidgetToSection(appWidgetID: number, targetSectionName: string,
targetIndex?: number): Promise<void>`, following the exact convention every other data-mutating
action in that file already follows (verified against the file's current shape, not an earlier
read, since it was concurrently edited today by the light-save and z-index-atomicity efforts):

- **Category B** (real API calls, not a pure client-side `layout` edit) — fires immediately, does
  not go through `mutateLayoutWithUndo`.
- Clears `undoStack`/`redoStack` in the same `set()` call (the file's own "CRITICAL FIX" convention:
  every untracked mutating action must invalidate pending undo/redo patches rather than risk a stale
  patch replaying against structurally-changed state).
- Bumps `structuralGeneration` (this is structural — a widget's section membership and the
  membership/order of every other widget in the target section can change).
- Uses the same "partial-failure, no rollback, apply-whatever-succeeded + throw a count" convention
  `reorderWidgetsInSection` already uses (not the heavier compensating-rollback convention
  `moveAppWidget`/`renameSection` use) — this action IS fundamentally a multi-widget `displayOrder`
  reshuffle within the target section plus one `sectionName` change, the same shape as
  `reorderWidgetsInSection`, not a two-widget atomic swap like `reorderCanvasWidgetZIndex`.

**Behavior**: looks up the moved widget's current section (no-ops if it's already in
`targetSectionName` — that's interaction #1's job, not this action's); builds the target section's
full post-insert ordering (existing siblings, by `displayOrder`, with the moved widget spliced in at
`targetIndex ?? siblings.length`); PUTs `{ sectionName: targetSectionName, displayOrder }` for the
moved widget and `{ displayOrder }` for every *sibling whose position actually shifted* because of
the insertion (same "only PUT what changed" rule `reorderWidgetsInSection` already follows — no
bulk-reorder endpoint exists on the backend). The source section is **not** compacted/renumbered —
`displayOrder` gaps are harmless (every read site only ever `.sort()`s by it, never assumes
contiguity), and `reorderWidgetsInSection`/`moveAppWidget` already establish that gaps are an
accepted, unfixed characteristic of this data model, not something this action needs to newly solve.

### 6.2 Insert position: append vs. insert-at-drop-position

**Decision: insert-at-drop-position when dropped on a specific chip, append-to-end when dropped on a
section's empty background.** Rationale: drag-and-drop is the one placement gesture in this whole
Designer that inherently communicates a desired position (that's the entire point of a spatial
drop), so ignoring it and always appending would throw away real signal the user is giving for free.
This mirrors `useSectionActions`' own existing same-section reorder logic, which already inserts at
the exact dropped-on chip's position rather than always appending — extending the same convention to
cross-section drops is the more consistent choice, not a new one. Dropping on a section's own empty
background/padding (not any specific chip — e.g. an empty section, or dropping below the last chip)
falls back to append, which is the only sane interpretation of "no specific chip was targeted."

### 6.3 Where drop targets live

Both existing `WidgetChip`/`useSectionActions` call sites get updated drop handling — no third
call site is being added for chip-level drag (see §5 of `03-...md`'s Blockers section, which already
flagged canvas-mode's Up/Down repurposing; this is unrelated, flow-mode-and-canvas-mode-agnostic):

- **`WidgetChip.tsx`**: gains a required `sectionName: string` prop (so its own `onDragStart` can
  stamp the drag payload with *its own* section, not just the widget ID) and its `onDrop` prop
  signature changes from `() => void` to `(e: React.DragEvent) => void` (so the receiving
  `useSectionActions` instance can read `e.dataTransfer` — the payload, not local component state,
  is now the source of truth for *which* widget is being dragged, since local `draggedID` state is
  only ever populated in the section where the drag *started*, not the one it's dropped *into*).
- **`useSectionActions.ts`**: `handleDrop` (drop on a specific chip) now decodes the payload; same
  `sourceSectionName` as this hook's own `sectionName` → existing reorder path (unchanged, just now
  keyed off the payload's ID instead of local state, which is an equivalent value for that case but a
  uniform code path). Different `sourceSectionName` → the new `moveAppWidgetToSection` call, inserted
  before the dropped-on chip (§6.2). Toolbox payloads dropped directly on a chip use that chip's own
  position as the insert point too — a chip is just a narrower target within the same drop surface,
  not a semantically different action for a toolbox item (§7).
- A new `handleContainerDrop` (drop on the section's own background, not a chip) is added alongside
  it, wired to a `onDragOver`/`onDrop` pair on `SectionCard.tsx`'s `widgetSlots` wrapper div and
  `SectionDetailsContent.tsx`'s Widgets accordion body — both already unconditionally render (even
  with zero widgets, showing the "No widgets" message), so an empty section is always a valid drop
  target. Both handlers are gated off entirely for `readOnlyWidgets` sections (page-controlled
  primary-content sections — matches `WidgetChip`'s own existing `readOnly` gate, which already
  disables drag/drop chrome for those).

**Known, deliberately deferred limitation**: a *collapsed* `SectionCard` (the chevron-toggle,
independent of `readOnlyWidgets`) doesn't render its `widgetSlots` div at all, so it isn't a valid
drop target while collapsed. Auto-expanding on `dragover` would be a reasonable follow-up but adds
real complexity (a timer, a way to tell "hovering to expand" from "just passing through") for a
minor case — explicitly not done this pass, not a silent gap.

**Not supported, and explicitly out of scope**: dragging a widget that's actually rendered in the
live WYSIWYG canvas (a real form, a real HTML block) out of its section by grabbing the rendered
content itself. That would require adding `draggable`+`dragstart` to `AppPlayer`'s `WidgetSlot` — the
exact shared-package boundary Task 3's own design doc already identified and deliberately avoided
crossing for its free-position drag. `WidgetChip` rows (Site Structure drawer, Section Details
popover) remain the only place an *already-placed* widget can be picked up and moved between
sections. Toolbox → canvas placement (§7) does *not* have this problem, because the *target* there
is a read-only DOM query (`[data-section]`), not a drag *source* inside `AppPlayer`.

## 7. Toolbox panel (§3, interaction #3)

### 7.1 Placement and shell

New "Toolbox" drawer, using the exact same `Drawer.tsx` primitive `PagesPanel`/`SiteStructurePanel`
already use (Task 2, Phase 4) — not a new panel type, not a permanent sidebar. `designer-ui.store.ts`'s
`DesignerDrawer` union grows from `'pages' | 'structure' | null` to `'pages' | 'structure' | 'toolbox'
| null`; `DesignerToolbar.tsx` gets a third icon button next to the existing Pages/Site Structure pair
(`FiBox`, "Toolbox — drag new or existing widgets onto any section"). Rejected alternative: a
permanent docked palette column (closer to some Wix-adjacent tools' literal layout) — rejected because
this codebase already settled on "drawers for anything that isn't the always-visible canvas itself"
as its own convention this session (Pages, Site Structure), and a fourth, fifth interaction surface
competing for permanent screen width would fight the unified-canvas philosophy Task 2 established.

### 7.2 Contents: "New" tab and "Existing" tab, reusing existing data sources exactly

Mirrors `AddWidgetModal`'s own two tabs, same underlying data:

- **New tab**: one row per `WIDGET_TYPE_REGISTRY` entry (`@app-studio/core`) — the exact same
  registry `AddWidgetModal`'s "New Widget" tab already iterates, so a future 6th `WidgetType` needs
  one registry entry, not a second hardcoded list. `NOT_YET_AVAILABLE` types (currently
  `workflow-template-category`, `chat-panel` — **re-verified fresh, immediately before this doc's own
  first edit to any shared file, per this task's collision-safety instructions**, since a separate
  concurrent effort was investigating registering new types into this exact set) render visibly
  disabled with a "Coming soon" badge, matching `AddWidgetModal`'s own honest-disabled precedent —
  not draggable, not clickable.
- **Existing tab**: `WidgetsApiClient.list()` — the same call `AddWidgetModal`'s "Existing Widget" tab
  already makes, fetched on tab-select the same way.

### 7.3 Two different placement behaviors within the "New" tab — a deliberate split, not an inconsistency

`buildNewWidgetConfiguration`'s switch (moved, unchanged, into a new shared module — see §7.4) has
two branches with a **hard-required** field the widget genuinely cannot render without: `form` needs
a `formId`, `workflow-template` needs an `executionTemplateID`. Every other branch (`content`,
`page-navigation`, `hil-inbox`, `site-branding`, `signin`, `notifications`) has only optional fields
with real, already-shipped defaults (`AddWidgetModal`'s own initial `useState` values).

A bare drag-and-drop gesture has no way to collect a required value mid-drag. Two options were
considered:

1. Instant-create every type on drop, leaving `form`/`workflow-template` with an empty/undefined
   required field.
2. Split behavior by whether the type has a hard-required field.

**(1) was rejected outright** — it would silently reproduce the exact bug class this codebase already
hit and fixed once (`AddWidgetModal.tsx`'s own doc comment: a previous version wrote the wrong config
keys for Form Widgets, so `FormWidgetHandler.render()` read `undefined` and every such widget
silently failed to render). `AddWidgetModal` itself already hard-blocks this
(`if (widgetType === 'form' && !formId) { setError('Select a form'); return; }`) — a Toolbox
shortcut that bypassed that exact validation would be a regression, not a feature.

**Decision (2)**: the Toolbox's "New" tab items split into two interaction styles based on whether
their type is in a new `SAFE_DEFAULT_WIDGET_TYPES` list (`content`, `page-navigation`, `hil-inbox`,
`site-branding`, `signin`, `notifications`):

- **Safe-default types**: genuinely draggable (`draggable`, real `dragstart`). Dropping one onto a
  section instant-creates the widget with the same default field values `AddWidgetModal`'s own
  initial state already uses, placed at the actual drop position (§6.2's insert-vs-append rule
  applies here too — unlike `AddWidgetModal`'s button click, which has no spatial position to
  read and keeps its own existing (unchanged) `displayOrder: 0` behavior). The widget is fully
  usable immediately and editable afterward via the same in-place `WidgetDetailsContent`/inline-Tiptap
  editing Task 2 already established — "place first, configure by clicking it" is this codebase's
  own already-shipped philosophy for content editing, just now extended to placement.
- **`form`/`workflow-template`**: NOT draggable (`draggable={false}`, cursor communicates
  click-not-drag). Clicking the row instead opens the existing `AddWidgetModal`, pre-scoped to
  whichever section is currently selected in the Designer (`selectedSectionKey`, falling back to the
  layout's first section) via a new, additive `initialWidgetType?: WidgetType` prop — pre-selects the
  New-tab radio and widget type, so the click genuinely saves the user a step (no re-picking the
  type) rather than being a no-op shortcut to the exact same modal state they'd land in anyway. If no
  section is selected at all (a user opens the Toolbox before ever clicking into the canvas), the
  click shows an inline "Select a section first" message instead of opening a modal scoped to nothing
  — same honest-error-over-silent-wrong-behavior precedent as everywhere else in this design.

This keeps the Toolbox's drop-handling code (§6.3, §7.5) never having to know about per-type
validation at all — it only ever receives drag payloads for types it can 100% safely instant-create,
because the Toolbox UI itself is what enforces that boundary at drag-start time, not the drop side.

### 7.4 Reuse, not duplication: `buildNewWidgetConfiguration` and the "place a WidgetDto" call

`buildNewWidgetConfiguration` (previously a local, unexported function inside `AddWidgetModal.tsx`)
and the "create a Widget then place it" / "place an existing Widget" two-call sequences move,
unchanged in logic, into a new shared module: `widgets/addWidgetActions.ts`. `AddWidgetModal.tsx`'s
own `handleAddNew`/`handleAssignExisting` are refactored to call the same shared functions instead of
inlining the API calls directly — this is the "reuse `buildNewWidgetConfiguration`'s existing logic
... reuse the existing 'place a WidgetDto' logic" the task explicitly asked for, and it also means
any future fix to either function (like the real `formId`/`mode` casing bug `AddWidgetModal`'s own
doc comment already documents once) only has one place to land, not two that can drift.
`AddWidgetModal`'s own two handlers are behaviorally **byte-identical** after this refactor (same
calls, same `displayOrder: 0`, same `refreshAppWidgets()`/`onClose()` sequence) — this is a pure
extraction, not a rewrite of working code.

### 7.5 Drop targets: three surfaces, one decode function

- **Site Structure drawer / Section Details popover** (`SectionCard.tsx`/`SectionDetailsContent.tsx`,
  via `useSectionActions`'s `handleDrop`/`handleContainerDrop`, §6.3) — a toolbox payload dropped here
  calls the new `createSafeDefaultWidgetInSection`/`placeExistingWidget` helpers (§7.4), then
  `refreshAppWidgets()` (same convention `AddWidgetModal` already uses — this is structural, not the
  content-only light-save path).
- **The live WYSIWYG canvas** (`LivePreviewPanel.tsx`) — a new optional
  `onWidgetDropOnSection?: (sectionName: string, dataTransfer: DataTransfer) => void` prop, wired via
  `onDragOver`/`onDrop` on the same wrapper `<div>` that already delegates `onClick`/`onPointerDown`
  via `[data-widget]`/`[data-section]` — reading `dataTransfer` at `drop` time and calling
  `event.target.closest('[data-section]')` to find the target section name. Zero `AppPlayer` changes
  (see §4) — this is the identical delegation technique the click-to-select handler right above it in
  the same file already uses, just for a `drop` event instead of a `click`. `CanvasPanel.tsx` supplies
  the callback, decoding the payload once (via `decodeDragPayload`, §5) and dispatching to
  `moveAppWidgetToSection` (widget-chip payloads — §7.6 covers why this is included) or the shared
  `addWidgetActions.ts` helpers (toolbox payloads), computing `displayOrder` as "current widget count
  in that section" (append) since the live canvas has no concept of "which specific chip you're
  hovering" the way the list surfaces do.

### 7.6 A cross-surface case included on purpose: chip-from-popover → canvas section

`SectionDetailsContent` renders inside `OverlayPanel` — a popover anchored near whatever was clicked,
*not* a full-screen scrim like `Drawer.tsx` (which does block the canvas underneath from receiving
drag events at all, making a drawer→canvas chip-drag physically unreachable — a non-issue, not a gap
being silently accepted). Because parts of the live canvas stay visible and interactive around an
open `OverlayPanel`, a user really can start a widget-chip drag from the Section Details popover and
drop it on a different, visible section in the canvas. Since `CanvasPanel`'s own drop handler already
calls the same `decodeDragPayload` used everywhere else, handling this case costs nothing extra
(it's the same `widget-chip` branch already needed for symmetry) — included deliberately rather than
special-cased away.

## 8. Data model impact

**None beyond the store action itself.** No new `AppWidgetDto`/`AppSection` field, no DB migration,
no backend endpoint change — every new capability here is built entirely out of `sectionName` +
`displayOrder` mutations through the existing `AppWidgetsApiClient.update`/`.create` calls, exactly
like every pre-existing mutating action in `app-selection.store.ts`. `AppWidget.sectionName` changes
(the cross-section move) and `displayOrder` recomputation (§6.1/§6.2) are the only data changes this
design makes, and both already have first-class support in the existing DTO/API surface (confirmed:
`AppWidgetsClient.update(appId, id, body: Partial<AppWidgetDto>)` already accepts both fields — see
`app-widgets.client.ts`).

## 9. Backward compatibility

Zero effect on any app that never touches drag-and-drop or the Toolbox drawer: the new drawer value
(`'toolbox'`) is additive to a union that already defaults to `null`; the new store action is never
called unless a user actually performs a drop; `WidgetChip`'s new `sectionName` prop and `onDrop`
signature change are internal to this package (not part of any external API surface) and every call
site is updated in the same pass; `AddWidgetModal`'s new `initialWidgetType` prop is optional and
defaults to today's exact behavior (`'form'` pre-selected) when omitted. No existing widget, section,
or app is touched by simply having this code present and unused.

## 10. `AddWidgetModal`'s click-through flow: kept, unchanged, as the primary entry point for two types

Per the task's own default: `AddWidgetModal` is **not removed or functionally changed**. It remains
reachable exactly where it always was (`SectionCard`'s "+" icon, `SectionDetailsContent`'s "Add
Widget" button) and is now *additionally* the Toolbox's own required path for `form`/`workflow-template`
(§7.3) — it goes from being the *only* way to add a widget to being the *primary* way for two types
and an *alternative* way (a user can still open it manually and skip dragging entirely) for every
other type. Nothing about its existing behavior, validation, or API calls changed — only its two
creation call-sequences were extracted into a shared module it now calls into (§7.4), which is a
refactor of its internals, not a behavior change.

## 11. Effort / verification

This is additive UI + one store action wired into three drop surfaces, not a new subsystem — no
multi-week estimate needed the way Task 3's canvas-positioning analysis required one. Real build,
verified via `pnpm -r typecheck`/`tsc --noEmit` across every touched package (no test tooling exists
in `app-studio-designer-components-react`/`app-studio-store-react`, confirmed earlier this session —
not introduced as a side effect of this task). **Live browser E2E verification is explicitly NOT done
this pass** — flagged in `00-implementation-status.md`, matching how every other feature in this
initiative has been tracked (build-and-typecheck now, one consolidated live pass later).
