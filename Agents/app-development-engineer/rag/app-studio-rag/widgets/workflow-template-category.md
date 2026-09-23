# `workflow-template-category` — Workflow Category

A grid of every agent in one `Process_ExecutionTemplateCategories` category — each rendered as its
own card with Execute/Chat Now, same as the singular `workflow-template` widget. **Modal-required.**
Source: `widget-handlers-workflow-template-category-widget\src\
WorkflowTemplateCategoryWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `executionTemplateCategoryID` | number | **Yes** | — | The one category this widget's grid is scoped to. |

## Example

```json
{ "executionTemplateCategoryID": 7 }
```

## Gotchas

- Each card in the grid behaves identically to the standalone `workflow-template` widget — see that
  doc for Execute/Chat Now/HIL/Executions-history behavior, which applies per-card here too.
