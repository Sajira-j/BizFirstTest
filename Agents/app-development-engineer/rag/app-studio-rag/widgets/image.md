# `image` — Image

A single photo — URL, alt text, optional caption, object-fit, and an optional link. **Modal-required**
(`imageUrl` has no sensible default). Source: `widget-handlers-image-widget\src\ImageWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `imageUrl` | string | **Yes** | — | Set via the Media Source picker (Browse Media Library / Enter Custom URL) — see `../app-model.md`/gotchas below, never a bare text box in the real UI. |
| `altText` | string | No | — | Auto-fills from the picked asset's name when using the Library path and left blank. |
| `caption` | string | No | — | Shown below the image if set. |
| `objectFit` | `'cover'\|'contain'\|'fill'` | No | `'cover'` | CSS `object-fit`. |
| `linkUrl` | string | No | — | Wraps the image in `<a href>` when set. |
| `presentationStyle` | `'plain'\|'framed'\|'polaroid'\|'ken-burns-hover'` | No | `'plain'` | Visual treatment preset — `ken-burns-hover` is a subtle continuous zoom while hovering (single-image cousin of the gallery widgets' always-on Ken Burns theme). |

## Example

```json
{ "imageUrl": "https://.../photo.jpg", "altText": "Team offsite 2026", "objectFit": "cover", "presentationStyle": "framed" }
```

## Gotchas

- No server-side thumbnail generation exists anywhere in this pipeline — `imageUrl` always serves
  the ORIGINAL uploaded file, regardless of display size. Don't assume a resized/thumbnail variant
  is available.
- Every image/gallery-image render has `loading="lazy"` applied — offscreen images defer loading,
  but this doesn't reduce the bytes transferred for images that ARE visible.
- Media Source picker only ever surfaces assets flagged `IsPublicAsset === true` — a hard backend
  boundary, not just a UI filter.
