# `pdf-gallery` — PDF Gallery

Same shared `GalleryWidgetConfig` shape as `image-gallery.md` — read that doc for the full field
table; this file covers what's DIFFERENT. Live query against public PDF documents. Source:
`widget-handlers-pdf-gallery-widget\src\PdfGalleryWidgetConfig.ts` (extends `GalleryWidgetConfig`).

## Additional field

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `displayMode` | `'link'\|'embed'` | No | `'link'` | `'link'` = each card is a styled link opening the PDF in a new tab (same default as singular `pdf`). `'embed'` = each card renders an inline `<embed>` viewer. |

## Difference: theme set

`theme`: only 2 of the 5 values are valid — `'classic-grid'\|'slide-carousel'`.

## `displayMode: 'embed'` — allowed in both layouts, sized differently

Unlike a naive assumption that embed-mode only makes sense in a single-item carousel, `embed` is
deliberately allowed in BOTH `classic-grid` (each grid cell gets its own smaller embedded viewer)
and `slide-carousel` (one full-size viewer at a time).

## Example

```json
{ "theme": "classic-grid", "displayMode": "embed", "columns": 2 }
```

## Gotchas

- Same zero-filter/security rules as `image-gallery.md` apply here, for PDF assets.
