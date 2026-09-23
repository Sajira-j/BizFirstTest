# `site-branding`

Source: `packages/widget-handlers-site-branding-widget`, `WidgetRecord.ts`'s own doc comment (dated
2026-08-28, "app-studio-widget-library-expansion"), scanned 2026-08-30.

A new App.Name/App.IconUrl display widget — renders the App's own name and icon (not a user-editable
content block; it reflects the App record directly). Useful for header/branding sections that should
always match the App's real identity without a builder having to re-type it.

## Config — OPEN QUESTION

Field shapes not independently re-verified this pass. Likely minimal (display-mode/size toggles only,
since the actual name/icon come from the App record itself, not from `configuration`). Re-grep the
handler package's own config type before generating config for this widget type.
