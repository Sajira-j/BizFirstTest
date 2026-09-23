# `video-gallery` — Video Gallery

Same shared `GalleryWidgetConfig` shape as `image-gallery.md` — read that doc for the full field
table; this file only covers what's DIFFERENT. Live query against public videos.

## Difference: theme set

`theme`: only 3 of the 5 `GalleryTheme` values are valid for video — `'classic-grid'\|
'fade-carousel'\|'slide-carousel'` (no `masonry`, no `ken-burns` — those are image-specific).

## Example

```json
{ "theme": "slide-carousel", "automation": "relaxed", "documentTypeID": 5 }
```

## Gotchas

- Same zero-filter/security/no-thumbnail rules as `image-gallery.md` apply here, for video assets.
