# `hil-inbox` — HIL Inbox

A human-in-the-loop actionable inbox — approvals, forms, and tasks routed to this app's users.
**No `configuration` at all** — auth/tenant scoping comes entirely from `WidgetRenderContext`, read
directly by the renderer. Source: `widget-handlers-hil-inbox-widget\src\HilInboxWidgetHandler.ts`.

## Config

None. `render()` returns `{ type: 'hil-inbox' }` with no fields.

## Example

```json
{}
```

## Gotchas

- Don't generate a `configuration` object for this widget type — there is nothing to set. If a
  request implies per-instance filtering/scoping for this widget, that capability doesn't exist
  today; flag it as a gap rather than inventing a config field.
