# `form` — Form Widget

Binds to one form defined in Atlas Forms (a separate, shared form-building system) via `formId`,
and renders it in one of four modes. The generic "any table, generic CRUD" building block — one
Form Widget placement = one way of interacting with one Atlas Forms-defined form/table.
**Modal-required** (not a safe-default drag-place type — `formId` has no sensible default).
Source: `widget-handlers-form-widget\src\FormWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `formId` | number | **Yes** | — | The Atlas Forms form to bind to. Pick via the FormID lookup control (search-as-you-type), never a bare ID. |
| `mode` | `'create'\|'edit'\|'view'\|'list'` | No | `'list'` (per creation UI) | `create` = FormRenderer 'edit' mode with empty initialData; `edit`/`view` map directly; `list` = a minimal record table (NOT FormRenderer), re-queries reactively on `filterVarKey` change, opens a row via `rowClickAction` into a paired Form Widget in `view` mode elsewhere. |
| `formOverrides.hideSearchFilterArea` | boolean | No | unset (form's own value) | CSS-hides every `role: 'filters'` section while keeping controls mounted/wired. |
| `formOverrides.autoSearchOnLoad` | boolean | No | unset | Fires the form's `search` action once automatically on mount. |
| `formOverrides.searchResultViewMode` | `'grid'\|'cards'\|'gallery'` | No | `'grid'` (no-op) | `cards`/`gallery` are real live-rendered modes — the target grid control still needs its own `cardTemplate` authored on its schema. |
| `submitActionBindings` | string | No | — | Post-submit action wiring. |
| `additionalConfig` | `Record<string,string>` | No | — | Freeform escape hatch, namespaced so it can never collide with typed fields; nothing reads it by default until code explicitly looks for a key. |
| `readOnly` | boolean | No | `false` | |
| `hideSubmitButton` | boolean | No | `false` | |
| `initialData` | `Record<string,unknown>` | No | — | |
| `filterVarKey` | string (list mode only) | No | — | App var whose value is used as the record-query filter. |
| `pageSize` | number (list mode only) | No | `20` | |
| `rowClickAction` | `ActionBinding[]` (list mode only) | No | — | Fired on row click, receives the clicked record's ID as `params.dataID`. |

## Example

```json
{ "formId": 30600, "mode": "list", "filterVarKey": "selectedProjectID", "pageSize": 20 }
```

## Gotchas

- One form definition can serve as a full search UI in one placement and a hidden-filter,
  auto-loaded list in another purely via `formOverrides` — no need to duplicate the form.
- `mode: 'list'` is NOT FormRenderer — it's a separate minimal table component. Don't assume List
  mode inherits FormRenderer's own field-level behavior.
- The "Configure Form" gear icon (next to the Form field in the edit UI) opens Atlas Forms' real
  Form Editor inline — the same gear-lookup pairing now reused across Atlas Forms' own pickers.
