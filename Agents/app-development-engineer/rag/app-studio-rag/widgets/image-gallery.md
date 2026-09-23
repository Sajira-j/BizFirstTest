# `image-gallery` — Image Gallery

A LIVE QUERY (not a fixed list) against public images, run at render time, displayed in one of 5
themes. Unlike `image` (one fixed URL), this widget re-queries the backend every render. Source:
`widget-handlers-gallery-shared\src\GalleryWidgetConfig.ts` (shared by all 4 gallery widgets),
`widget-handlers-image-gallery-widget\src\ImageGalleryWidgetConfig.ts` (= `GalleryWidgetConfig`
directly, no additions), `widget-handlers-gallery-shared\src\themes.ts`.

## Config (shared shape — `GalleryWidgetConfig`)

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `title` | string | No | — | |
| `documentTypeID` | number \| null | No | `null` (no filter) | Real searchable lookup in the UI, not a bare ID box. |
| `documentCategoryID` | number \| null | No | `null` (no filter) | Same. |
| `mediaCategory` | string | No | an implied default per widget kind (image widgets default to an image category), overridable | |
| `theme` | `GalleryTheme` | Yes (set by the creation wizard) | — | For image-gallery: `'classic-grid'\|'masonry'\|'fade-carousel'\|'slide-carousel'\|'ken-burns'` (all 5). |
| `automation` | `'off'\|'relaxed'\|'standard'\|'energetic'` | No (carousel themes only) | `'standard'` | Named preset, not raw seconds. |
| `columns` | number | No (grid-style themes only) | — | |
| `maxItems` | number | No | unlimited | |

## Example

```json
{ "theme": "masonry", "columns": 3, "documentCategoryID": 12, "maxItems": 24 }
```

## Gotchas

- **Zero filters set = shows ALL public images** — a deliberate design goal, not a bug.
- Only `IsPublicAsset === true` assets are ever queryable here — hard backend boundary.
- No server-side thumbnail generation — every item downloads the original file even at a small tile
  size (`loading="lazy"` is applied, which defers offscreen loads but doesn't reduce bytes for
  visible items).
- This is a LIVE query, not a snapshot — the same rendering appears identically in Preview and real
  App Player, re-evaluated each time.
