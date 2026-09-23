# Target: App Studio — per-section region placement + widget-stacking flex model

## Problem statement

App Studio's `AppLayout.type` (`sidebar-left`/`sidebar-right`/`top-nav`/`top-nav-sidebar`/
`full-width`/`custom`) looks like a real region system but isn't one — confirmed by reading the
actual renderer (`app-handlers-generic/src/AppPlayer.tsx`, `getLayoutContainerStyle`): it just
picks ONE global flex direction (row for sidebar-left/right, column for top-nav) for the *entire
app*, and every `AppSection` in `layout.appSections` is rendered as a flat, ordered flex child in
that single direction. There is no way today to get a left sidebar + top header + main content
simultaneously, and no per-section control over how ITS OWN widgets stack (currently hardcoded
`flex-wrap` in `SectionCard`'s designer view; the real `app-player` runtime has no equivalent
control at all).

Binoy wants:
1. A real per-section **region** — header / left / right / main / footer — so multiple regions
   can coexist in one layout, not just one global axis.
2. A per-section **widget-stacking flex model** — direction (row/column), wrap, gap, alignment —
   so different sections can arrange their own widgets differently (a nav section as a tight
   horizontal row, a content section as a wrapped grid, etc.).
3. Both exposed as real UI: a region picker on **Add Section**, and the flex controls on **Edit
   Section** — using `react-icons/fi` (Feather, the only icon set this codebase uses anywhere),
   not a plain `<select>`:
   - Region: Header → `FiArrowUp`, Left → `FiArrowLeft`, Right → `FiArrowRight`,
     Main/Content → `FiSquare`, Footer → `FiArrowDown`
   - Widget direction: Row (side-by-side) → `FiColumns`, Column (stacked) → `FiList`
   - Render as an icon-button group with the active option highlighted (same visual pattern
     already used for the move-up/down/add/delete row in `SectionCard`'s header), not a
     dropdown.

## Design (already worked out — implement this, don't redesign from scratch)

### Data model — `app-handlers-core/src/types/AppSection.ts`

Add two new **optional** fields (default behavior must exactly match today's — a section with
neither field set renders exactly as it does now, so every existing app in the DB keeps working
unchanged):

```ts
export interface AppSection {
  // ...existing fields unchanged...
  /** Where this section sits in the layout's region grid. Omitted = 'main' (today's only
   *  behavior — a flat, ordered stack). */
  region?: 'header' | 'left' | 'right' | 'main' | 'footer';
  /** How this section's OWN widgets stack. Omitted = today's default (row, wrap, existing gap). */
  widgetLayout?: {
    direction?: 'row' | 'column';
    wrap?: boolean;
    gap?: number;
    align?: 'start' | 'center' | 'end' | 'stretch';
    justify?: 'start' | 'center' | 'end' | 'space-between';
  };
}
```

### Renderer — `app-handlers-generic/src/AppPlayer.tsx`

Replace the single `getLayoutContainerStyle(layout.type)` flat-flex-child approach with a real
CSS Grid: define named grid areas (`header`/`left`/`right`/`main`/`footer`) via `grid-template-areas`,
group `layout.appSections` by `section.region ?? 'main'` into those areas, and render each
region's sections stacked within their own area (order within a region still follows array order,
same as today). `layout.type` becomes a *preset* that pre-fills sensible defaults for a
2-column/3-column/header-only shape (keep it — don't remove the existing enum, it's still useful
as a starting template) but per-section `region` is the actual source of truth for where a
section renders, overriding the preset's assumptions where a section explicitly sets one.

Backward compatibility is non-negotiable here: every section in every existing app has
`region: undefined` today. Confirm (with a real test against existing app data, e.g. app 1052)
that the new renderer produces IDENTICAL output for a layout where no section sets `region` —
this is the acceptance test for Phase 1, not optional polish.

Apply `section.widgetLayout` to that section's widget container instead of the current hardcoded
flex-wrap style — same backward-compat requirement: `widgetLayout: undefined` must render
identically to today's fixed behavior.

### Designer UI

- **`CanvasPanel.tsx`'s `handleAddSection`**: add a region picker (the 5-icon group described
  above) to the Add Section flow — a small inline picker, not a separate modal, matching how
  lightweight the rest of that action already is. Default selection: Main/Content (`region`
  omitted from the new section, matching today's behavior exactly).
- **`properties/SectionEditor.tsx`** (the Properties-overlay section editor) and/or the
  `CanvasPanel.tsx` `SectionDetailsContent` inline panel (today's rename UI lives in BOTH places
  per the same-day 2026-08-25 rename work — check whether the region/widget-layout controls
  belong in one, the other, or both, and keep them consistent with wherever rename ended up
  living predominantly): add the region picker (5-icon group) and the widget-layout controls
  (direction 2-icon group + wrap toggle + gap number input + align/justify selects).
- Reuse `ui/IconButton.tsx` for every icon-group option, matching the exact selected/unselected
  visual treatment already established this session (see `SectionCard.tsx`'s move-up/down
  buttons for the pattern: `disabled` state, `title` tooltip, consistent sizing).

## Constraints (apply throughout)

- No auto-commits/pushes.
- Additive-only to `AppSection`/`AppLayout` — do not change or remove any existing field, do not
  require either new field. This is what makes the migration free (no DB migration, no backfill).
- No premature abstraction — five region values and five flex properties is the actual, current
  ask; don't build a generic arbitrary-grid-position system "for the future."
- `pnpm typecheck`/`pnpm build` clean for every touched package/app. `apps/app-studio-designer`
  and `apps/app-player` were both confirmed building 100% clean on 2026-08-25 after a separate
  build-blocker fix pass — do not regress that.
- Dated `DevelopmentHistoryLog.md` entries in every touched package.
- Read `app-studio/docs/QA-Session-2026-08-25.md` first for full same-day context (a widget
  registration-parity check between the designer's `useLivePreviewEngine.ts` and `app-player`'s
  `App.tsx` is already a known, separate concern noted there — don't let this target regress that
  parity either, since both files render sections and both need the new region-aware logic kept
  in sync).
- Verify live in the browser against a real app (1052) before considering this done — a layout
  change is exactly the kind of thing that can typecheck clean and still look wrong.
