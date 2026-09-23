# `hil-inbox`

Source: `packages/widget-handlers-hil-inbox-widget`, `WidgetRecord.ts`'s own doc comment (dated
2026-08-28, "app-studio-widget-library-expansion"), scanned 2026-08-30.

One of 4 thin-wrapper widgets added together that each consume an existing shared BizFirst UI
package rather than reimplementing UI — real package directory confirmed. Backs `@bizfirst/hil-ui` +
`@bizfirst/hil-react`'s Human-in-the-Loop inbox/notification UI (approval queues, HIL action items)
embedded as an App Studio widget.

## Config — OPEN QUESTION

Field shapes not independently re-verified this pass. Re-grep the handler package's own config type
(likely filters by App/tenant scope, possibly a max-items or view-mode setting) before generating
config for this widget type.
