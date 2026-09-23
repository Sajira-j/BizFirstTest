# App Studio — Schema Overview

Source of truth: `app-studio` monorepo (`packages/app-handlers-core/src/types/`), scanned 2026-08-30.
Always load this file whole; see `01-common-properties.md` for fields shared by every section/widget
and `widgetTypes/{widgetType}.md` for widget-specific `configuration` properties.

## Top-level shape

```
App (AIExt_Apps)
 ├─ Configuration (JSON string, parsed to { meta: AppMeta, layout: AppLayout })
 ├─ AppPage[]   (AIExt_AppPages)
 └─ AppWidget[] (AIExt_AppWidgets) — placement rows, each pointing at one shared Widget
```

## AppLayout

```ts
interface AppLayout { type: AppLayoutType; appSections: AppSection[]; hideMobileHamburger?: boolean; }
type AppLayoutType = 'sidebar-left' | 'sidebar-right' | 'top-nav' | 'top-nav-sidebar' | 'full-width' | 'custom';
```

## AppSection — recursive, see `01-common-properties.md` for the full field table

```ts
interface AppSection {
  name: string;                          // required, unique within its own siblings array
  appSections?: AppSection[];            // RECURSIVE — a section can contain child sections
  region?: 'header'|'left'|'right'|'main'|'footer';
  widgetLayout?: AppSectionWidgetLayout;  // flex config for this section's own widgets only
  sectionContainer?: StyleSlotValue;      // see 02-style-properties.md
  sectionBackground?: StyleSlotValue;
  isPrimaryContentSection?: boolean;      // marks the section whose widgets can be page-scoped
  icon?: string; iconUrl?: string;
  responsive?: { tablet?: Record<string,string>; mobile?: Record<string,string> };  // legacy, superseded by 02's design
  hiddenOn?: ('mobile'|'tablet'|'desktop')[];
}
```

**A section with no `appSections` children and no `region` set behaves exactly as the oldest, simplest
apps in this system always have** — a flat, unstyled, stacked block. Every additive field above is
opt-in with an unchanged-behavior default when omitted — this is a deliberate, consistent convention
across this whole type system, not accidental.

## AppPageRecord

| Field | Type | Notes |
|---|---|---|
| `appPageID` | number | PK |
| `parentPageID` | number\|null | self-referencing FK; drives an expandable page-tree widget (`page-navigation`) |
| `name`, `title`, `slug` | string | `slug` used in URLs (`/{appCode}/page/{slug}`) |
| `pageType` | `'Physical'\|'Virtual'` | |
| `pageCategory` | `'DataPage'\|'ContentPage'` | |
| `appWidgetID` | number\|null | deprecated, not removed — pre-page-first-design pages set this; new pages leave it null and use `AppWidget.appPageID` instead (see below) |
| `entityType` | string\|null | only for `pageType==='Virtual'` |
| `showInMenu`, `menuLabel`, `menuPosition` | | |
| `menuTargetType` | `'Page'\|'ExternalUrl'\|'Anchor'\|null` | polymorphic menu-item click target, matches Wix/Webflow/Squarespace's own menu-item type selector |
| `externalUrl`, `openInNewWindow` | | only when `menuTargetType==='ExternalUrl'` |
| `anchorTarget` | string\|null | only when `menuTargetType==='Anchor'` — matched against a rendered `[data-section="..."]` attribute, scrolls on the CURRENT page, never navigates |
| `isDefault`, `isActive`, `displayOrder` | | |
| `seo` | `AppPageSeo?` | per-page SEO override, falls back to the App's own `meta` — see field table below |

### AppPageSeo (per-page, optional — falls back to App-level `meta` when unset)

| Field | Type | Notes |
|---|---|---|
| `metaTitle` | string? | falls back to `AppPageRecord.title`, then `App.meta.title` |
| `metaDescription` | string? | falls back to `App.meta.description` |
| `ogImageUrl` | string? | |
| `canonicalUrl` | string? | falls back to the computed route URL for this page |
| `robotsIndex`, `robotsFollow` | boolean | default `true` |
| `structuredData` | string? | raw JSON-LD, escape hatch only |

## AppWidgetRecord — the placement row

| Field | Type | Notes |
|---|---|---|
| `appWidgetID` | number | PK of this placement |
| `widgetID` | number | FK to the shared `Widget` definition (name/type/configuration are on the Widget, not here — see below) |
| `sectionName` | string | which `AppSection.name` this widget lives in |
| `displayOrder` | number | ordinal position within its section — NO free-position x/y exists today (see `architecture.md` §8) |
| `appPageID` | number\|null | **page-scoping**: `null` = shared/default content, rendered when no page is targeted; a real ID scopes this widget to exactly one Page. Only meaningful inside the section flagged `isPrimaryContentSection` |
| `isDefault` | boolean | true = default for this section; `widgetID=0` in a URL resolves here |
| `showInNav`, `navPosition` | | `navPosition: 'top'\|'side'\|null` |
| `routable` | boolean | true = update the browser URL when this widget is active |
| `configuration` | object\|null | **placement-level override** of the Widget's own base `configuration` — see `01-common-properties.md` |
| `styleConfiguration` | `WidgetStyleConfig?` | current, structured style (see `02-style-properties.md`); `widgetStyle`/`widgetCss` are the legacy flat fields, superseded but not removed |
| `widget` | `WidgetRecord` | the joined shared Widget definition |

## WidgetRecord — the shared, reusable widget DEFINITION (one row can be placed many times)

| Field | Type | Notes |
|---|---|---|
| `widgetID` | number | PK |
| `name`, `description` | string | |
| `widgetType` | `WidgetType` | one of the real types below — determines which `IWidgetHandler` renders it |
| `configuration` | object | the widget's own base config — see `widgetTypes/{widgetType}.md` for the real shape per type |
| `isSystem`, `isActive`, `displayOrder` | | |

## WidgetType — the real, current list (verify by re-grepping `packages/widget-handlers-*-widget` before trusting this if picking this up much later — it grows)

`form` | `content` | `workflow-template` | `workflow-template-category` | `chat-panel` |
`page-navigation` | `hil-inbox` | `signin` | `notifications` | `site-branding`

One `.md` file per type under `widgetTypes/` in this same `rag/v1/` folder.
