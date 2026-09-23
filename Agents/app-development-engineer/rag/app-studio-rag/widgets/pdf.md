# `pdf` — PDF

A single PDF — URL, optional title, shown as a link or an inline embed. **Modal-required** (`pdfUrl`
has no sensible default). Source: `widget-handlers-pdf-widget\src\PdfWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `pdfUrl` | string | **Yes** | — | Set via the Media Source picker. |
| `title` | string | No | falls back to a generic "View PDF" | Link text / accessible name. |
| `displayMode` | `'link'\|'embed'` | No | `'link'` | `'link'` = styled new-tab link (safe default, doesn't grow the page tall). `'embed'` = inline `<embed>` viewer. |

## Example

```json
{ "pdfUrl": "https://.../report.pdf", "title": "Q3 Report", "displayMode": "embed" }
```
