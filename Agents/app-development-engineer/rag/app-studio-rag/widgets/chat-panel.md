# `chat-panel` — Chat Panel

An embedded chat window for triggering and conversing with one Process — new conversations only;
resuming a past one is read-only for now. **Modal-required** (`processID` has no sensible default —
click-to-configure, never drag-to-instant-place). Source: `widget-handlers-chat-panel-widget\src\
ChatPanelWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `processID` | number | **Yes** | — | `Process_Processes.ProcessID` — which process a new conversation triggers. |
| `executionTemplateID` | number | No | — | Threaded through to the first `triggerNewConversation` call only; the server hydrates the execution's InputData from this template. |
| `enableConversationList` | boolean | No | `true` | Chrome opt-out for the conversations icon + slide-out panel. |
| `welcomeTitle` / `welcomeSubtitle` | string | No | — | |

## Example

```json
{ "processID": 12, "executionTemplateID": 42, "enableConversationList": true }
```

## Gotchas

- Resuming a past conversation through this widget is read-only today — don't generate a flow
  expecting to continue an old conversation interactively here.
