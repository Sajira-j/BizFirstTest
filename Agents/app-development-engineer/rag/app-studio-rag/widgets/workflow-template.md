# `workflow-template` — Workflow Agent

Binds to one real `Process_ExecutionTemplates` row via `executionTemplateID`, renders a card with
that agent's name/description/icon plus an **Execute** or **Chat Now** button — which one shows is
driven entirely by the bound template's own **Type** (must be a conversational type), never
something configured on the widget itself. **Modal-required.**
Source: `widget-handlers-workflow-template-widget\src\WorkflowTemplateWidgetConfig.ts`,
`bizfirst-common\ai-agent-catalog-ui\src\components\TemplateCard\`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `executionTemplateID` | number | **Yes** | — | The one agent this widget renders. Best set via the Toolbox's "Workflow Agents" drag-list (lists templates by real name), not the modal's bare numeric field. |

## Example

```json
{ "executionTemplateID": 42 }
```

## Runtime behavior (not config, but essential to generating a correct workflow around this widget)

- **Chat Now is a full-page redirect to ChatDesk**, a separate app — by design, not an inline
  window.
- **Execute** resolves to one of three real outcomes: success with no HIL input needed, a
  synchronous inline HIL Form/Chat render right on the card (real, working — see Gotchas), or a
  visible error.
- An **Executions** history popover shows past runs (status/time/error) and auto-highlights the
  just-triggered run — there is no standalone single-execution detail page anywhere in this
  codebase to link to instead (a future FlowInsights app is the intended eventual target — see the
  `buildExecutionDetailUrl` prop below).
- `TemplateCard`'s `buildExecutionDetailUrl?: (executionID: string) => string | undefined` prop is
  a swappable, currently-unset integration point for that future single-execution page.

## Gotchas

- Execute-vs-Chat-Now is driven by the template's **Type** field, NOT its Category — a template
  that "should" chat but shows Execute almost always means its Type field is unset/wrong upstream
  in Flow Studio/WorkDesk, not a bug in this widget.
- Synchronous inline HIL rendering after Execute uses the universal `HilPresentationRequestEnvelope`
  contract and the shared `@bizfirst/hil-ui`/`hil-ui-atlas-forms` stack — the same rendering
  pipeline Flow Studio's own execution preview uses, not a separate implementation. Any HIL reply
  submitted through it uses real authenticated submission (`ConfiguredFetchHilSubmitClient`) — a
  prior bug where replies silently failed due to an unauthenticated/relative URL was found and
  fixed; don't reintroduce a bare/unconfigured submit client if touching this code.
