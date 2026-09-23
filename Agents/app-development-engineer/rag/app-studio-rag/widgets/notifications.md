# `notifications` — Notifications

A notifications bell with unread count and dropdown list. Safe-default. Source:
`widget-handlers-notifications-widget\src\NotificationsWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `viewAllUrl` | string | No | omitted → no "View all" link shown | Optional link at the bottom of the dropdown. |

## Example

```json
{ "viewAllUrl": "/notifications" }
```
