# `workflow-template`

Source: `WidgetRecord.ts`'s own doc comment + `WidgetRenderResult`'s discriminated union
(`app-handlers-core`), scanned 2026-08-30. **Config field shapes below are NOT independently verified
against the handler package's own source this pass** — confirmed only at the render-result-shape
level; re-grep `packages/widget-handlers-workflow-template-widget/src` before trusting field names.

Renders a single AI-agent execution-template card via `@bizfirst/ai-agent-catalog-ui`'s `TemplateCard`
component (reused, not reimplemented) — resolution of the ID to a real `ExecutionTemplateDTO`
(name/description/icon/Execute-or-Chat-Now) happens inside the paired React `Component`, not in
`render()` itself (mirrors how the `form` type only carries `formId`, never the fetched schema, for
the same reason — defer live data fetching to the mounted component).

## Render result shape (confirmed)

```ts
{ type: 'workflow-template', executionTemplateID: number }
```

## Config (OPEN QUESTION — not independently re-verified this pass)

At minimum carries an execution-template ID (backing the render result above). Re-check the real
`WidgetConfig` type in the handler package's own source before generating config for this widget
type.
