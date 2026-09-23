# `form`

Source: `packages/widget-handlers-form-widget/src/FormWidgetConfig.ts`, scanned 2026-08-30. Embeds a
real Atlas Forms form via `FormRenderer` (reused from the `atlas-forms` monorepo, not reimplemented).

## Config

| Property | Type | Notes |
|---|---|---|
| `formId` | number | required — the target `Atlas_Forms` row |
| `mode` | `'create'\|'edit'\|'view'\|'list'` | `create`/`edit` both map to `FormRenderer`'s `'edit'` mode (`create` = empty `initialData`); `view` maps to `FormRenderer`'s `'view'`; `list` is NOT `FormRenderer` at all — a minimal record table that reactively re-queries on `filterVarKey` changes and opens a row via `rowClickAction` into a paired form widget elsewhere in the layout, in `view` mode |
| `readOnly` | boolean? | |
| `hideSubmitButton` | boolean? | |
| `initialData` | `Record<string,unknown>?` | |
| `filterVarKey` | string? | **`list` mode only** — the App var whose value is used as the search filter |
| `submitActionBindings` | string? | |

## Render result shape

```ts
{ type: 'form', formId, mode: 'edit'|'view', props: { readOnly, hideSubmitButton, initialData } }
```
(`'create'` is resolved to `'edit'` with empty `initialData` before this point — never appears in the
render result itself.)

## Examples

Create a new record:
```json
{ "formId": 42, "mode": "create" }
```

Read-only detail view (typically paired with a `list`-mode form elsewhere via `rowClickAction`):
```json
{ "formId": 42, "mode": "view", "readOnly": true }
```

List/search table:
```json
{ "formId": 42, "mode": "list", "filterVarKey": "searchQuery" }
```
