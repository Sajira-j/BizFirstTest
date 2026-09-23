# `page-navigation`

Source: `packages/widget-handlers-page-navigation-widget`, `WidgetRecord.ts`'s own doc comment,
scanned 2026-08-30. **Config field shapes not independently re-verified this pass beyond what's
below** — real package directory confirmed to exist.

Renders an end-user, freely-placeable menu of `AppPage`s — expandable/collapsible, tree-shaped via
each `AppPageRecord.parentPageID`. This is the real Widget counterpart to the legacy fixed
`AppNavMenu` top/side chrome bars: `AppPlayer.tsx` has a `hasPageNavigationWidget` gate — once an app
places one of these widgets, the old fixed nav bar stops rendering (this session found and fixed a
real bug where that gate didn't also account for single-page apps that build their own nav a
different way — see `lessons/README.md` in the wix-style-design project, not this one, for the
qoboto-specific fix).

## Config (OPEN QUESTION — field names not independently re-verified this pass)

Almost certainly includes at minimum: which pages to include/exclude, orientation (top/side, mirrors
`AppPageRecord.menuPosition`), and whether to respect `AppPageRecord.showInMenu` per page. Re-grep the
handler package's own config type before generating config for this widget.
