# Common Properties — Sections and Widgets

Properties shared across every `AppSection`/`AppWidgetRecord` regardless of widget type.
Widget-type-specific `configuration` fields live in `widgetTypes/{widgetType}.md`. Styling: see
`02-style-properties.md`.

## Section-level shared properties (`AppSection`, `app-handlers-core/src/types/AppSection.ts`)

| Property | Type | Default when omitted | Notes |
|---|---|---|---|
| `name` | string | — required | join key `AppWidgetRecord.sectionName` points at; renaming a section must cascade-update every widget's `sectionName` (this codebase's own store does this atomically — see `architecture.md` §2) |
| `appSections` | `AppSection[]?` | none — flat leaf | **recursive** — always walk with `sectionTree.ts`'s helpers, never hand-rolled flat-array logic |
| `region` | `'header'\|'left'\|'right'\|'main'\|'footer'?` | implicit `'main'`, legacy flex/flow container | setting `region` on even ONE section in a layout switches the whole renderer to a named-region CSS Grid |
| `widgetLayout` | `AppSectionWidgetLayout?` | plain stacked block children, no flex container at all | `{direction?:'row'|'column', wrap?, gap?, align?, justify?}` |
| `isPrimaryContentSection` | boolean? | `false` | marks the ONE section whose widgets can carry a page-scoping `appPageID` |
| `sectionContainer` / `sectionBackground` | `StyleSlotValue?` | unstyled | Level 2 style slots |
| `icon` / `iconUrl` | string? | none | |
| `hiddenOn` | `('mobile'|'tablet'|'desktop')[]?` | visible everywhere | |

## Widget-placement shared properties (`AppWidgetRecord`)

| Property | Type | Default | Notes |
|---|---|---|---|
| `sectionName` | string | — required | must match a real `AppSection.name` |
| `displayOrder` | number | — required | ordinal only; no x/y coordinate field exists in this data model today |
| `appPageID` | number\|null | `null` | shared/default content unless set AND the widget's section is `isPrimaryContentSection` |
| `isDefault` | boolean | `false` | |
| `showInNav` / `navPosition` | boolean / `'top'|'side'|null` | `false` / `null` | |
| `routable` | boolean | `false` | |
| `configuration` | object\|null | inherits the Widget's own base `configuration` when `null` | placement-level OVERRIDE — merges over (does not replace outright, per each handler's own `load()`) the shared Widget's base config |
| `styleConfiguration` | `WidgetStyleConfig?` | unstyled | current structured style mechanism — 10 named sub-regions, only `widgetContainer`/`widgetBackground` wired into the renderer today |

## Widget definition shared properties (`WidgetRecord` — the reusable definition, not a placement)

| Property | Type | Notes |
|---|---|---|
| `widgetType` | `WidgetType` | determines dispatch — see `00-overview.md` |
| `configuration` | object | base config, shape is 100% widget-type-specific — see `widgetTypes/{widgetType}.md`, never guess a shared shape here |
| `isSystem` | boolean | system-provided widgets vs. user-created |
| `isActive` | boolean | |

## The `IWidgetHandler` contract every widget type implements

```ts
interface IWidgetHandler {
  readonly widgetType: string;
  load(widget: WidgetRecord): Promise<void>;
  render(ctx: WidgetRenderContext): WidgetRenderResult;   // pure data resolution, no DOM/React access
  unload(widgetId: string): Promise<void>;
  readonly Component?: ComponentType<WidgetHandlerRenderProps>;  // mounts render()'s result
}

interface WidgetRenderContext {
  widget: WidgetRecord;
  appWidget: AppWidgetRecord;
  appID: number; tenantID: number; isStudio: boolean;
  resolveToken: (template: string) => string;
  getAuthToken: () => string | Promise<string>;  // the ONLY sanctioned way a widget Component reaches the session token
}
```

`render()` returns a small discriminated-union `WidgetRenderResult` per type (e.g.
`{type:'content', format, sanitizedHtml}`, `{type:'form', formId, mode, props}`) — never the widget's
own raw `configuration`. Dispatch is a plain `registry.get(widgetType)` lookup — see `architecture.md`
§1 for why this matters for anything generating or modifying App Studio content programmatically.

## Minimal valid widget placement

```json
{
  "sectionName": "hero",
  "displayOrder": 1,
  "isDefault": false,
  "showInNav": false,
  "navPosition": null,
  "routable": false,
  "configuration": null
}
```

(`configuration: null` inherits the shared Widget's own base config unchanged — only set a
placement-level override when this specific instance genuinely needs to differ.)
