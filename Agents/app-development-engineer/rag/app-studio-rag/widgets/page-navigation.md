# `page-navigation` — Page Navigation

A placeable, end-user menu of an App's `AppPage`s, tree-shaped via `parentPageID` (nesting) —
expandable/collapsible. The real Widget counterpart to the legacy fixed `AppNavMenu` top/side bars,
which stop rendering once an app places one of these (`AppPlayer.tsx`'s `hasPageNavigationWidget`
gate). Source: `widget-handlers-page-navigation-widget\src\PageNavigationWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `orientation` | `'vertical'\|'horizontal'` | No | `'vertical'` | Vertical = column, accordion-style expand/collapse. Horizontal = row, dropdown-style expand/collapse beneath the parent. |

## Example

```json
{ "orientation": "horizontal" }
```

## Gotchas

- Placing this widget REPLACES the legacy fixed nav bars for the whole app, not just adds an
  alternative — if an app already relies on the legacy bars, adding this widget changes that app's
  navigation UX globally.
- Nesting comes entirely from `AppPageRecord.parentPageID` — this widget doesn't have its own
  separate page-tree config; it reads the app's real page hierarchy.
