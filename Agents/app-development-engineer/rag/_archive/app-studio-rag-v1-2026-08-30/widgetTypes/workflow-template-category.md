# `workflow-template-category`

Source: `WidgetRecord.ts`'s own doc comment, scanned 2026-08-30. **Real finding**: this session's
directory scan found no dedicated `workflow-template-category` package either (same 9-package list
noted in `chat-panel.md`) — likely implemented inside `widget-handlers-workflow-template-widget`
alongside `workflow-template` itself (same package, two registered types), but NOT confirmed. Verify
`registry.register('workflow-template-category', ...)` actually exists before relying on this type
being usable.

Renders a GRID of multiple `ExecutionTemplateDTO` agent cards (the multi-item counterpart to the
single-card `workflow-template` type above) — same "defer the live fetch to the mounted `Component`,
`render()` only carries identifying data" pattern.

## Render result shape (by naming convention with `workflow-template` — not independently confirmed this pass)

Almost certainly a category/filter identifier (e.g. a category ID or tag) rather than a single
`executionTemplateID` — OPEN QUESTION, verify against real source before relying on this.
