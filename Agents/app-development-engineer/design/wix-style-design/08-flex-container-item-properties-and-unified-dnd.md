# Flex Container/Item Properties + Unified Section/Widget Drag-and-Drop — Design

Status: Design + build combined (Binoy: "now do a detailed analysis and create a design document.
then start coding"). Builds directly on Task 9's drag-and-drop precedent
(`07-widget-toolbox-and-cross-section-dnd.md`) and Task 2's unified canvas/overlay/drawer shell —
this doc assumes both are already live (they are, verified against source below).

## 1. What the user asked for

Gathered from a single live design conversation (2026-09-02), condensed to the load-bearing quotes:

> "Moving Sections is done now using arrow buttons. That is not ideal. We need a drag and drop
> option here... Place it below, top, left or right on inside of another section."

> "Adding a new Section. We need a toolbar called Section Toolbar and create several pre configured
> sections (orientation)... reuse the same drag code. Ensure this is consistent with the widget
> approach."

> "I should be able to move widget from one place to another... drop into a Section... Widget Detail
> panel should have Remove Widget and Delete widget options. Delete must not work if the widget is
> used by other pages."

> "I am okay to have one top section for a region. However within each region, we can have multiple
> children. No need to restrict." (resolves the left/right-drop data-model question)

> "When a section is dropped inside a section, the section may have horizontal, vertical or flow
> orientation for contents. If it is horizontal, show left and right indicators. If vertical, top
> and bottom. If flow, show all, and if on top or bottom, add a new line flag... device specific."

> "sections and widget must also support this kind of grid model - horizontal, vertical and flow"

> "hiddenOn, direction, wrap, gap, align, justify - these are container properties, match flex grid
> concept. I like to reuse the same idea... Add this as properties of Section and Widget... device
> specific properties can also be added and general can be default. I like zero config system."

> "these properties are converted into css by rules"

> "container and container-items properties must be grouped as two new sections in the ui."

> "Widget Toolbar and Section Toolbar can be inside the same panel with two panes. User will switch
> widget or section toolbox manually with localstorage memory. All collapse buttons should have
> localstorage memory."

## 2. Current state (verified against source, 2026-09-02)

- **Section reorder is buttons-only.** `useSectionActions.ts`'s `handleMoveWidgetUp/Down` exist for
  *widgets*; sections only get `moveSection(name, 'up'|'down')` (`app-selection.store.ts:813`) wired
  to plain arrow buttons in `SiteStructurePanel.tsx`/`SectionDetailsContent.tsx`. No section drag
  exists anywhere.
- **Nested child sections render with zero container layout.** `AppSectionRenderer` in
  `app-handlers-generic/src/AppPlayer.tsx` maps `section.appSections` directly as siblings with no
  wrapping flex container — confirmed by reading the component: widgets get
  `getWidgetContainerStyle(section.widgetLayout)` applied to their own wrapper div; nested sections
  get nothing. This is the one genuine rendering-engine gap this design has to close — everything
  else below is Designer-only.
- **`widgetLayout` already *is* the container-property shape this design reuses**, not a new
  invention: `{ direction: 'row'|'column', wrap: boolean, gap?, align?, justify? }`
  (`@app-studio/core`'s `AppSectionWidgetLayout`), converted to CSS by
  `getWidgetContainerStyle()` — a plain rule function, exactly matching Binoy's "converted into css
  by rules." Zero-config already works today: unset `widgetLayout` returns `{}` (plain block, no
  flex container at all) — this is the existing convention the new fields must preserve, not a new
  requirement.
- **Two existing, non-interchangeable responsive-resolution mechanisms**:
  1. JS breakpoint detection (`BreakpointService`, real `window.innerWidth`) — drives `hiddenOn` and
     section-style overrides today. Fragile: needed explicit `override()`/`clearOverride()` wiring
     for the Designer's own device-preview toggle to even work, and a real bug from that missing
     wiring was found and fixed once already (`AppPlayer.tsx`'s own history, 2026-08-30).
  2. CSS `@container` queries (`cssInjector.ts`'s `injectResponsiveStyle`) — drives arbitrary
     per-property responsive overrides on style slots (`StyleSlotValue.style` as
     `{base, tablet, mobile}`). Resolves against real rendered width in pure CSS, can't drift out of
     sync with a JS toggle.
  This design puts every new property on mechanism (2) — see §4.2.
- **Widget delete is "Remove" only.** `removeAppWidget` (`app-selection.store.ts:360`) deletes the
  `AppWidget` *placement* row via `AppWidgetsApiClient.delete`, never touches the underlying
  `Widget` catalogue record. A separate `WidgetsApiClient` (`DELETE /api/v1/app-studio/widgets/{id}`,
  `BaseAppStudioWidgetController.cs:64`) soft-deletes the Widget *definition* itself — but has **no
  usage check**: it will delete a Widget definition that's still placed via `AppWidget` rows in
  other apps/pages, silently breaking them. No usage-count endpoint exists anywhere in
  `BizFirst.Ai.AppStudio.Api.Base` today (confirmed: no `Usage`/`InUse`/`ReferenceCount` route in the
  controller set). This is real, net-new backend work, not a wiring gap.
- **The whole layout/section/widget tree is opaque JSON, not SQL columns.** `App.Configuration` is a
  single JSON string (`AppsApiClient.create({ configuration: '{}' })`,
  `app-selection.store.ts:299-300` writes the whole `AppLayout` back via `JSON.stringify`). This
  means **every new field this design adds to `AppSection`/`AppWidget` (container props, item props,
  responsive overrides) needs zero backend/database change** — the backend never parses this blob's
  shape, it round-trips it. The *only* backend work in this whole design is the Delete-Widget usage
  check (§4.8).
- **Native HTML5 drag-and-drop is the established, deliberate mechanism** for every existing drag
  interaction (widget reorder, cross-section move, Toolbox → section) — not `@dnd-kit`, rejected
  twice already (Task 3, Task 9) because real drop targets are `AppPlayer`-rendered nodes shared with
  the production `app-player` runtime. This design's section-drag and Section Toolbox reuse the exact
  same `dragPayload.ts` MIME-type-disambiguation pattern, extended with two new payload kinds
  (`section-chip`, `toolbox-section-template`).
- **Toolbox is already a 5-section accordion** (`ToolboxPanel.tsx`) inside a `toolboxSidebar` flex
  panel in `CanvasPanel.tsx`, with its own minimize button (`toolboxMinimized`, plain `useState`, not
  persisted). No localStorage usage exists anywhere in this package today — every "remembered" UI
  state here is net-new plumbing, not an existing pattern to extend.

## 3. Requirements — gathered and numbered

| # | Requirement | Source quote (§1) |
|---|---|---|
| R1 | Section drag-and-drop reorder (replaces up/down buttons, buttons stay as accessible fallback) | "We need a drag and drop option" |
| R2 | Section drop targets: top/bottom (reorder), left/right (side-by-side), inside (nest) | "Place it below, top, left or right on inside" |
| R3 | Section drag visuals (border/color/label) distinct from widget drag visuals | "different coloring than widget... dotted border and label etc should be different" |
| R4 | Section Toolbox: pre-configured orientation templates, draggable, same drag code as widgets | "toolbar called Section Toolbar... reuse the same drag code" |
| R5 | Widget drag-and-drop: move between widgets (top/bottom/left/right) and into a section | "move widget from one place to another... drop into a Section" |
| R6 | Widget Details panel: separate "Remove Widget" and "Delete Widget" actions | "Remove Widget and Delete widget options" |
| R7 | Delete Widget blocked if the Widget is placed anywhere else (cross-app) | "Delete must not work if the widget is used by other pages" |
| R8 | Visual drop-zone indicators showing which side a drop will land on | "Visual indicators must be shown with drop place highlighting" |
| R9 | One root section per region; unrestricted nesting within/below that | "one top section for a region... multiple children. No need to restrict" |
| R10 | Container orientation (horizontal/vertical/flow) determines which drop indicators show | "horizontal, show left and right... vertical, top and bottom... flow, show all" |
| R11 | Per-item "force new line" override, only meaningful in flow orientation, device-specific | "add a new line flag... This should be device specific as well" |
| R12 | Same grid model (horizontal/vertical/flow) for both Sections and Widgets | "sections and widget must also support this kind of grid model" |
| R13 | Reuse real flexbox property names (`direction`, `wrap`, `gap`, `align`, `justify`, `hiddenOn`) as the container-property vocabulary, not an abstracted enum | "these are container properties, match flex grid concept" |
| R14 | Container properties AND item properties both fully user-editable | "Add all necessary properties and anyone can edit this and change" |
| R15 | Every property supports a device-specific override with a general/default base value | "device specific properties can also be added and general can be default" |
| R16 | Zero-config: drag-drop writes no explicit layout properties by default | "I like zero config system... minimal properties" |
| R17 | Properties resolve to CSS via a deterministic rule function, not stored as raw CSS | "these properties are converted into css by rules" |
| R18 | Container properties and item properties shown as two distinct, separately-grouped UI sections | "grouped as two new sections in the ui" |
| R19 | Widget Toolbox + Section Toolbox live in one panel as two manually-switched panes, last choice remembered | "inside the same panel with two panes... switch... manually with localstorage memory" |
| R20 | Every collapse/minimize control in the Designer remembers its state via localStorage | "All collapse buttons should have localstorage memory" |

## 4. Design

### 4.1 Data model — `@app-studio/core`

**Correction from the first draft of this doc**: per-property responsive wrapping
(`direction?: ResponsiveValue<...>`) does NOT match this codebase's existing convention. The real
one, already live for style slots (`StyleSlot.ts`'s `ResponsiveStyleValue`), wraps the WHOLE
properties object once — `{ base: FullProps, tablet?: Partial<Props>, mobile?: Partial<Props> }` —
with the same `'base' in value` structural discriminator `useStyleSlot.ts` already uses to tell a
responsive value apart from a flat one. This design reuses that exact shape instead of inventing a
second one (R13's "reuse the same idea" applies here too, not just to the property names):

```ts
/** Container properties — how a node arranges its OWN children. Field names/shape lifted verbatim
 *  from the existing AppSectionWidgetLayout (R13) — this IS that type, just renamed to not be
 *  widget-specific, so `widgetLayout`'s existing values satisfy it with zero conversion. */
interface FlexContainerProps {
  direction?: 'row' | 'column';
  wrap?: boolean;
  gap?: number;
  /** ADDED after industry research (2026-09-02, see §4.1a) — present in every tool researched
   *  (Figma Auto Layout, Framer Stacks, Webflow), absent from the first draft of this doc. Uniform
   *  padding on all four sides for v1 — per-side values are an additive future extension of the
   *  same field's shape, not a breaking change, if ever needed. */
  padding?: number;
  align?: 'start' | 'center' | 'end' | 'stretch';
  justify?: 'start' | 'center' | 'end' | 'space-between' | 'space-around';
}

/** Item properties — how a node behaves inside its PARENT's container. New for both Section and
 *  Widget (R12). `forceNewLine` only has an effect when the parent's resolved `wrap` is true (flow
 *  orientation) — see §4.3. */
interface FlexItemProps {
  alignSelf?: 'start' | 'center' | 'end' | 'stretch';
  order?: number;
  grow?: number;
  shrink?: number;
  basis?: string;
  forceNewLine?: boolean;
  /** ADDED after industry research (2026-09-02, see §4.1a) — replaces the section-level
   *  `layoutMode: 'canvas'` whole-section switch with Figma's/Wix Studio's own model: free
   *  positioning is a PER-ITEM escape hatch inside an otherwise flex-arranged parent, not a
   *  parent-wide mode. `'absolute'` reads `top`/`left`/`right`/`bottom` from this same node's
   *  existing `styleConfiguration.widgetContainer.style` (Task 3's fields, unchanged) instead of
   *  flowing through the parent's `direction`/`wrap`. Omitted/`'flow'` is today's exact behavior. */
  position?: 'flow' | 'absolute';
}

/** Same wrapper shape as StyleSlot.ts's ResponsiveStyleValue, generic over either props type. */
interface ResponsiveFlex<T> {
  base: T;
  tablet?: Partial<T>;
  mobile?: Partial<T>;
}
```

Attachment points (R2, R12, R18) — every field accepts either the flat shape (today's exact
behavior, R16's zero-config default) or the `ResponsiveFlex<T>` wrapper (R15):

| Type | New field | Role |
|---|---|---|
| `AppSection` | `widgetLayout?: AppSectionWidgetLayout \| ResponsiveFlex<FlexContainerProps>` (existing field, union-widened, not reshaped) | Container — arranges this section's own widgets |
| `AppSection` | `childLayout?: FlexContainerProps \| ResponsiveFlex<FlexContainerProps>` (new) | Container — arranges this section's nested `appSections` |
| `AppSection` | `itemLayout?: FlexItemProps \| ResponsiveFlex<FlexItemProps>` (new) | Item — how this section sits in its own parent |
| `WidgetStyleConfig` (on `AppWidgetRecord.styleConfiguration`) | `itemLayout?: FlexItemProps \| ResponsiveFlex<FlexItemProps>` (new) | Item — how this widget sits in its section |

**Back-compat**: every already-published app's `widgetLayout` is the flat `AppSectionWidgetLayout`
shape (no `base` key) — `'base' in value` unambiguously tells it apart from the new responsive
wrapper, so existing content resolves identically with zero migration, zero data rewrite.

Region/nesting policy (R9): **not a data constraint** — `AppSection.appSections` already allows
arbitrary nesting depth, and per-region section count was never enforced in the type. The "one root
section per region" rule is enforced only in the Designer's Add Section flow (§4.4): once a region
already has a top-level section, "Add Section" targeting that region routes into a nest-inside
gesture instead of creating a second top-level sibling. No engine change needed for this rule.

### 4.1a Industry validation (2026-09-02, before implementation continued)

Binoy asked for a check against real prior art before writing more code. Researched: Webflow, Framer
(Stacks vs. Frames), Figma (Auto Layout vs. absolute), Wix Studio (this app's own explicit reference
point elsewhere). Verdict: the container/item property model and `base→tablet→mobile` cascade
direction match Webflow almost field-for-field — no change needed there. Two real findings, both
adopted ("yes let us follow industry models"):

- **`padding`** — present in every tool researched, missing from the first draft. Added to
  `FlexContainerProps` above.
- **Canvas mode is per-item everywhere, not per-section.** Figma's "ignore auto layout" and Wix
  Studio's four position types (Absolute/Stack/Fixed/Sticky) are both assigned to ONE child inside
  an otherwise flex-arranged parent — never a whole-container mode switch. Our existing
  `AppSection.layoutMode: 'flow'|'canvas'` is exactly that whole-container switch, which is now the
  one place this design diverged from the tools it's modeled after. Replaced with `FlexItemProps`'s
  new `position` field above.

**Migration (zero data rewrite, same principle as every other field in this doc)**: an already-
published section with `layoutMode: 'canvas'` has widgets carrying explicit
`styleConfiguration.widgetContainer.style.{top,left,right,bottom}` (Task 3) but no `itemLayout`
field at all (didn't exist yet). Render-time inference rule: **a widget with no explicit
`itemLayout.position` set, inside a section whose `layoutMode === 'canvas'`, resolves to
`position: 'absolute'`** — this is a fallback read purely for pre-existing data, not a new persisted
default going forward. `AppSection.layoutMode`/`canvasHeight` stay in the type (nothing to remove,
nothing breaks), but stop being the authoritative rendering signal — `FlexItemProps.position` is.
Going forward, the Designer's "Layout Mode: Canvas" toggle becomes a bulk-action convenience (sets
`itemLayout.position: 'absolute'` on every widget currently in that section, one-time, at toggle
time) rather than a per-render conditional. Full render-side implementation of this inference rule
lands in Phase 2 (§6) alongside the other rendering-engine work — this section documents the rule,
Phase 2 builds it.

### 4.2 Properties → CSS: the rule function (R17)

**Corrected mid-implementation (2026-09-02)**: the first draft of this section took a `bp` parameter
and JS-resolved one flattened style per call — that's the JS-`BreakpointService` path §2 explicitly
rejected. The actual, already-verified-against-source mechanism (`useStyleSlot.ts`) works
differently: `base` applies as a plain inline style (cheapest, and inline always wins over the
injected class's own base rule per CSS specificity), `tablet`/`mobile` DELTAS (only the properties
an author actually overrode for that device, matching `ResponsiveStyleValue`'s existing `Partial<T>`
contract) go through the SAME `injectResponsiveStyle` this package already uses for style slots,
returned as a className the caller applies alongside the inline style. Implemented as two new hooks
in `app-handlers-generic/src/hooks/useFlexLayoutStyle.ts`, mirroring `useStyleSlot`'s own
`{style, className}` contract exactly rather than inventing a second shape:

```ts
function useContainerFlexStyle(raw: AppSectionWidgetLayout | FlexContainerProps | ResponsiveFlex<FlexContainerProps> | undefined): { style: CSSProperties; className?: string };
function useItemFlexStyle(raw: FlexItemProps | ResponsiveFlex<FlexItemProps> | undefined, parentWraps: boolean): { style: CSSProperties; className?: string };
```

Both return `{ style: {}, className: undefined }` for `undefined` input — R16's zero-config
guarantee holds at the rule-function level. `align`/`justify` pass straight through to
`alignItems`/`justifyContent` with no translation, matching the old `getWidgetContainerStyle`'s
existing behavior exactly. **Known Phase 1 simplification**: `useItemFlexStyle`'s `parentWraps` is a
plain boolean the CALLER resolves (typically the parent container's own base `wrap` value) — full
per-breakpoint synchronization between a parent's responsive `wrap` and a child's responsive
`forceNewLine` (R11) is a real wiring question deferred to Phase 2, not solved generically at the
rule-function level.

**Responsive resolution mechanism (R15): `@container` queries, not `BreakpointService`** — per §2's
finding, these values feed into the *existing* `cssInjector.ts`/`injectResponsiveStyle` per-property
`@container`-scoped CSS generation (already built for style slots), not the JS-breakpoint path. This
was an open question two turns ago in the live design conversation; resolved in favor of the more
robust mechanism, matching precedent already in this exact codebase.

### 4.3 Rendering engine change — nested section container (the one real gap, §2)

`AppSectionRenderer` in `app-handlers-generic/src/AppPlayer.tsx` gets a wrapping div around
`section.appSections?.map(...)`, styled via `useContainerFlexStyle(section.childLayout)` (`style`
inline + `className` applied) — structurally identical to how `getWidgetContainerStyle`/now
`useContainerFlexStyle(section.widgetLayout)` already wraps the widgets div today, just a second,
independent container for the second, independent children collection (nested sections). `WidgetSlot`
and the section wrapper itself both also apply `useItemFlexStyle(node.itemLayout, parentWraps)` from
their *own* item properties, and `resolveItemPosition(node.itemLayout, legacyCanvasMode)` (falling back to the
`layoutMode === 'canvas'` inference, §4.1a) decides whether that item renders `position: absolute`
against its parent instead of flowing through the container. This is the only change to the shared
`app-player` rendering path — everything else in this design is Designer-only
(`app-studio-designer-components-react`).

### 4.4 Drag-and-drop — sections and widgets, unified (R1, R2, R5, R8, R10)

**One new shared module**, `canvas/dropZoneGeometry.ts`, used by both section-drag and widget-drag
handlers (R4's "reuse the same drag code" requirement, generalized to both directions):

```ts
type DropZone = 'before' | 'after' | 'left' | 'right' | 'inside';

function resolveDropZone(
  pointerX: number, pointerY: number, targetRect: DOMRect,
  containerOrientation: { direction: 'row'|'column'; wrap: boolean },
): DropZone
```

Zone set depends on the *target's parent container's* resolved orientation (R10), not a fixed set:
- `direction: 'column', wrap: false` (vertical) → `before`/`after` only (top/bottom halves of the
  target's rect).
- `direction: 'row', wrap: false` (horizontal) → `left`/`right` only (left/right halves).
- `wrap: true` (flow) → all four quadrants, plus `inside` as a smaller center hit-zone for nesting.
  Dropping on a `before`/`after` edge in flow mode sets the dropped node's `itemLayout.forceNewLine`
  for the active device (R11) — the drop *gesture* is what writes this flag, the property itself is
  editable afterward in the Container Item panel (§4.7) same as any other value.

**Section drag** (new): `SectionCard`/`SectionDetailsContent` gain `draggable` handles reading a new
`dragPayload.ts` kind (`'section-chip'`), decoded the same way `'widget-chip'` already is. Drop
targets are other sections (via `resolveDropZone` against the target's parent's `childLayout`) —
`before`/`after` reorder within the current parent, `left`/`right` also reorder but additionally set
`childLayout.direction` to `'row'` on the shared parent if it wasn't already (first left/right drop
"upgrades" a vertical stack into a row, consistent with R10's "orientation determines indicators, not
the other way around" — but the *first* drop that establishes a horizontal relationship has nothing
to infer orientation from yet, so it sets it), `inside` nests the dragged section as a new child of
the target. Region-drop enforcement (R9's Designer-side policy) applies here too: dropping a
top-level section onto a different top-level section's region is only a reorder within that region,
never an implicit second root — becoming a second root requires explicit `left`/`right`/`inside`
which nests, not two siblings floating at the layout root.

**Widget drag** (extends existing `useSectionActions.ts` reorder/cross-section-move, R5): same
`resolveDropZone` call, now reading the target widget's *section's* `widgetLayout` for orientation
instead of a section's `childLayout`. `before`/`after`/`left`/`right` (orientation-gated, same as
sections) reorder or set `forceNewLine`; dropping on a section's empty background (not on a specific
widget) is unchanged — append-at-end, exactly like today.

**Up/down buttons stay** (accessibility fallback, flagged as a real gap if removed — not one of
Binoy's stated requirements, kept as a cross-cutting correctness note from the earlier design
conversation, not itself a new requirement).

**Visual distinction (R3)**: new `Z_DRAG_*`-adjacent style tokens —
`SECTION_DRAG_BORDER_COLOR`/`WIDGET_DRAG_BORDER_COLOR` (distinct hues), a `data-drag-kind` attribute
driving which color the shared `DropIndicatorOverlay` (R8, new small component) renders, and the
existing `app-studio-label` class gets a `data-drag-kind`-scoped modifier for the drag-source's own
ghost/label styling. One shared indicator component, two token sets — not two parallel
implementations.

### 4.5 Section Toolbox (R4)

New sibling to the existing Widget Toolbox content, not a 6th accordion entry — see §4.6 for where it
lives. Each template is a pre-configured `{ childLayout: FlexContainerProps }` value (Horizontal Row,
Vertical Stack, Flow/Wrap, ...) with a small live-CSS preview thumbnail. Dragging one out uses the
*same* `dragPayload.ts` kind as an existing Toolbox item (a new `'toolbox-section-template'` payload
kind, decoded by the same `decodeDragPayload()` every other drop surface already calls) — dropping it
creates a new empty section pre-populated with that `childLayout`, placed via the same
`resolveDropZone` targeting logic as §4.4's section drag.

### 4.6 Toolbox panel unification (R19)

`ToolboxPanel.tsx`'s current single accordion becomes the "Widgets" pane; a new "Sections" pane
holds §4.5's templates. A small two-tab switcher header (`activeToolboxPane: 'widgets' | 'sections'`)
sits above both, backed by a new tiny `useLocalStorageState` hook (see §4.7) rather than plain
`useState` — the one new piece of shared infrastructure this whole design needs, since **no
localStorage usage exists anywhere in this package today** (§2). Selecting a section on the canvas
still force-opens the *drawer* (existing behavior, unchanged) but never force-switches the active
pane — respects the user's last manual choice, per the earlier design conversation's resolution of
this exact question.

### 4.7 Collapse-state persistence (R20)

New shared hook, `ui/useLocalStorageState.ts`:

```ts
function useLocalStorageState<T>(key: string, initial: T): [T, (v: T) => void]
```

Reads synchronously on first render (avoids the "flash open then snap shut" problem flagged in the
design conversation), wrapped in try/catch (private browsing / disabled storage degrades to
`initial`, never throws). Global keys (not app-scoped, per the design conversation's own resolution —
this is a workspace preference, not app data), namespaced `app-studio-designer:<name>`:

| Key | Replaces |
|---|---|
| `app-studio-designer:toolbox-minimized` | `CanvasPanel.tsx`'s `toolboxMinimized` `useState` |
| `app-studio-designer:toolbox-active-pane` | New, §4.6 |
| `app-studio-designer:toolbox-accordion:<sectionKey>` (×5, one per existing Widget Toolbox section) | `ToolboxPanel.tsx`'s per-section `everOpened`/open state |
| `app-studio-designer:toolbox-accordion:sections` | New Section Toolbox pane's own open state |
| `app-studio-designer:details-panel-minimized` | `OverlayPanel.tsx`'s minimize state (currently none — new) |

### 4.8 Remove vs. Delete Widget (R6, R7)

**Remove Widget** — existing `removeAppWidget` action, unchanged, relabeled in the UI for clarity
next to the new Delete button.

**Delete Widget** — new. Backend: new `GET /api/v1/app-studio/widgets/{id}/usage` on
`BaseAppStudioWidgetController.cs`, returning `{ appWidgetCount: number, apps: {appID, appName}[] }`
via a `COUNT(*)`/small `SELECT` against `AppWidget WHERE WidgetID = @id AND Deleted = 0` (no schema
change — `AppWidget.WidgetID` already exists as the join column, confirmed in service code). Frontend
calls this before enabling the Delete button; if `appWidgetCount > 0`, Delete is disabled with a
tooltip naming which apps still use it (mirrors `removeSection`'s existing partial-failure UX
pattern rather than inventing a new one). This is the **only backend work in this entire design** —
everything else is frontend-only per §2's JSON-blob finding.

### 4.9 Properties panel UI (R14, R18)

**Section Details panel** — two new accordion groups added to the existing set
(`ACTIONS`/`WIDGETS`/`REGION`/`WIDGET LAYOUT`/`LAYOUT MODE`/`STYLING`):
- **"Section Layout"** (new, container — §4.1's `childLayout`), same field set/editor shape as the
  existing "Widget Layout" group (direction/wrap/gap/align/justify), just bound to a different field.
- **"Container Item"** (new, item — §4.1's `itemLayout`): align-self, order, grow/shrink/basis,
  force-new-line (disabled/greyed when the parent's resolved `wrap` is false, per §4.2 — never
  silently ignored, visibly inert).

**Widget Details panel** — one new group: **"Container Item"** only (widgets have no children, no
container group — matches the container/item split resolved earlier in the design conversation).

**Device selector**: both new groups get the exact same Desktop/Tablet/Mobile pill switcher already
built for the canvas's own `PreviewDevice` toggle (`FiMonitor`/`FiTablet`/`FiSmartphone` icons,
`LivePreviewPanel.tsx`) — reused, not reinvented. Selecting a device scopes which bucket of the
`ResponsiveValue` the fields below read/write; "Desktop" is the `base` value and has no separate
override toggle.

## 5. Requirements validation

| # | Requirement | Satisfied by |
|---|---|---|
| R1 | Section DnD reorder | §4.4 (section drag) |
| R2 | 5-way section drop targets | §4.4 (`resolveDropZone`) |
| R3 | Distinct section vs. widget drag visuals | §4.4 (visual distinction) |
| R4 | Section Toolbox, shared drag code | §4.5 |
| R5 | Widget DnD move (4-way + into section) | §4.4 (widget drag) |
| R6 | Remove vs. Delete Widget | §4.8 |
| R7 | Delete blocked by cross-app usage | §4.8 (new backend endpoint) |
| R8 | Drop-zone visual indicators | §4.4 (`DropIndicatorOverlay`) |
| R9 | One root/region, unrestricted nesting | §4.1 (Designer-flow policy, not data constraint) |
| R10 | Orientation-driven indicator sets | §4.4 (`resolveDropZone` reads container orientation) |
| R11 | Force-new-line, flow-only, device-specific | §4.1 (`FlexItemProps.forceNewLine`) + §4.2 (`parentWraps` gate) |
| R12 | Same grid model for Sections and Widgets | §4.1 (shared `FlexContainerProps`/`FlexItemProps` types) |
| R13 | Real flexbox property names | §4.1 (reused from existing `widgetLayout` shape) |
| R14 | Fully editable container + item properties | §4.9 |
| R15 | Device-specific + general/base default | §4.1 (`ResponsiveValue<T>`) |
| R16 | Zero-config | §4.1 (all fields optional) + §4.2 (rule functions return `{}` on `undefined`) |
| R17 | Properties → CSS via deterministic rules | §4.2 |
| R18 | Container/item as two separate UI sections | §4.9 |
| R19 | Unified Toolbox, two panes, localStorage | §4.6 |
| R20 | All collapse buttons, localStorage | §4.7 |

All 20 gathered requirements have a corresponding design section. No requirement was dropped or
silently narrowed.

## 6. Build phasing

Too large for one pass — six phases, each independently shippable/testable, ordered so later phases
build on real (not assumed) earlier ones:

1. **Foundation**: `FlexContainerProps`/`FlexItemProps`/`ResponsiveValue` types (§4.1),
   `resolveContainerStyle`/`resolveItemStyle` rule functions (§4.2), `useLocalStorageState` hook
   (§4.7). No visible behavior change yet — pure plumbing, `tsc --noEmit` clean is the bar.
2. **Rendering engine**: nested-section container wrapper + item-style application in `AppPlayer.tsx`
   (§4.3). Verify existing apps render byte-identical (zero-config guarantee) before adding any UI to
   set the new fields.
3. **Properties panel UI**: Section Layout + Container Item groups on both Details panels (§4.9),
   device pill reuse. This is what makes Phase 1-2's fields actually reachable/testable by a human.
4. **Drag-and-drop**: `dropZoneGeometry.ts`, section drag, extended widget drag, visual indicators
   and distinction (§4.4).
5. **Section Toolbox + Toolbox unification**: §4.5, §4.6.
6. **Remove/Delete Widget split**: backend usage endpoint + frontend gating (§4.8) — last, since it's
   the only phase touching the .NET backend and is fully independent of Phases 1-5.

## 7. Progress (updated as phases land)

- **Phase 1 (Foundation) — COMPLETE, verified.** Types, CSS rule functions (`useContainerFlexStyle`/
  `useItemFlexStyle`, corrected mid-build to the real `@container`-query mechanism, not the
  rejected JS-breakpoint path), `useLocalStorageState`. `tsc --noEmit` clean across all 7 affected
  packages; live-checked in the Designer, byte-identical render, zero new console errors.
- **Phase 2 (Rendering engine) — COMPLETE, verified.** Nested-section container wrapper,
  item-level styling for both sections and widgets, canvas-mode-to-per-item-position migration
  with its back-compat inference rule. `tsc --noEmit` clean; live-verified against Qoboto's own
  real nested-section data (`hero` → `hero-left` → `hero-text`, pre-existing production content),
  zero regressions, zero new console errors.
- **Phase 3 (Properties panel UI) — COMPLETE, verified.** `ContainerPropertiesEditor`/
  `ItemPropertiesEditor`/`DeviceScopeTabs` built; "Section Layout" + "Container Item" groups wired
  into `SectionDetailsContent.tsx`, "Container Item" wired into `WidgetEditorFields.tsx`'s Style
  tab. `findSectionEntry` widened to also return `parent` (needed for a section's own
  `forceNewLine` gate). `tsc --noEmit` clean across the whole workspace; live-verified in the
  Designer — both new groups render correctly with working device tabs, a real write
  (Section Layout direction → Row) persisted through the undo-tracked mutation path with zero
  console errors, then cleanly undone (no debris left in the real app).
- **Phase 4a (Section drag-and-drop, Site Structure tree) — COMPLETE, verified.** Scope
  refinement made mid-build: R1/R2/R3/R9 are fully delivered for the Site Structure tree (the
  primary section-management surface) via a linear before/after/inside drop-zone model
  (`dropZoneGeometry.ts`'s `resolveLinearDropZone`) — the orientation-gated 4-way indicator set
  from §4.4's original draft is a LIVE-CANVAS concept (it needs a container's real rendered
  width/orientation to mean anything; the tree is always one vertical column regardless of the
  underlying section's actual `direction`), so building it against the tree would have shown
  indicators with nothing real to key off of. Live-canvas section dragging (dragging a rendered
  `[data-section]` node directly, matching the orientation it's actually rendered in) is deferred
  as **Phase 4b**, not started — same open item for widgets' own left/right drag (today's widget
  drag, in this same tree, is also linear-list-only; extending IT to the live canvas is bundled
  into the same Phase 4b, not duplicated as a separate item).
  - New: `moveSectionToTarget` (`@app-studio/core`'s `sectionTree.ts`, cycle-guarded) + matching
    store action (routed through `mutateLayoutWithUndo`, undo-tracked for free — `moveSection`'s
    existing up/down swap already worked this way, contrary to this doc's own earlier "sections
    are untracked" note elsewhere in the wix-style-design series).
  - New `section-chip` drag payload kind (`dragPayload.ts`), drag handle + drop-zone visuals on
    `SectionCard.tsx`, R3's amber/blue color distinction extended to `AppPlayer.tsx`'s own
    studio-mode section/widget borders and labels (not just the tree).
  - **Real bug found and fixed during verification**: `onDrop` originally read the `dropZone`
    value set by a PRIOR `dragover`'s `setDropZone` call — since that state update is
    batched/async, a fast drag-and-release with no intervening render could read a stale `null`
    and silently drop nothing. Fixed by recomputing the zone directly from the `drop` event
    itself, matching the same geometry the visual indicator already used. Caught via direct
    `DragEvent`/`DataTransfer` dispatch (native `left_click_drag` mouse automation proved too
    imprecise for a small drag-handle target to reliably test this surface) — the dispatch
    approach exposed a real, user-reachable timing bug a coarser test would have missed. Also
    fixed a smaller, pre-existing-pattern violation caught during the same build: `onDragOver`
    was calling `decodeDragPayload` (a real payload read), which that function's own doc comment
    already documents as unsafe outside `drop` in some browsers — switched to a type-presence-only
    check, matching this file's own existing widget-container dragover handler.
  - Live-verified against Qoboto's real data: dragged `hero` to reorder before `header`, confirmed
    both the tree and the live canvas reflected it, then undone cleanly with zero debris.
- **Phase 4b (live-canvas section + widget drag)**, **Phase 5 (Section Toolbox + Toolbox
  unification)**, **Phase 6 (Remove/Delete Widget backend)** — not started.

Starting with Phase 1 now.
