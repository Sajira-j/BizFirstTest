# `audio-gallery` — Audio Gallery

Same shared `GalleryWidgetConfig` shape as `image-gallery.md` — read that doc for the full field
table; this file only covers what's DIFFERENT. Live query against public audio tracks.

## Difference: theme set

`theme`: only 2 of the 5 `GalleryTheme` values are valid for audio — `'classic-grid'\|'slide-carousel'`.

## Example

```json
{ "theme": "classic-grid", "columns": 2 }
```

## Gotchas

- Same zero-filter/security rules as `image-gallery.md` apply here, for audio assets. Audio has no
  visual thumbnail concept at all (unlike image/video), so `classic-grid`/`slide-carousel` items
  render as players/labels, not thumbnails.
