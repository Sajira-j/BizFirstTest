# `notifications`

Source: `packages/widget-handlers-notifications-widget`, `WidgetRecord.ts`'s own doc comment (dated
2026-08-28, "app-studio-widget-library-expansion"), scanned 2026-08-30.

Thin wrapper consuming `@bizfirst/notifications-react`'s bell/notification-list UI as an embeddable
App Studio widget (the same `useNotifications` hook `AppShell.tsx` uses for the Designer's own
notifications bell — see `architecture.md`).

## Config — OPEN QUESTION

Field shapes not independently re-verified this pass. Re-grep the handler package's own config type
(likely a max-items/unread-only toggle, and a `viewAllUrl`) before generating config for this widget
type.
