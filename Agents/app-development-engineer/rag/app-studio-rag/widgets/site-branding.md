# `site-branding` — Site Branding

The App's own logo and name, side by side. Safe-default. Source:
`widget-handlers-site-branding-widget\src\SiteBrandingWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `titleOverride` | string | No | shows the App's real `Name` (fetched live) | A later App rename reflects automatically unless overridden. |
| `imageSize` | number | No | — | Logo height in pixels; width follows the image's own aspect ratio. |

## Example

```json
{ "imageSize": 40 }
```
