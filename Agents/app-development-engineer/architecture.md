# App Studio Automation — Architecture

Status: research/documentation, written 2026-08-30 as the first output of this project, based on the
qoboto-clone + "wix-style-design" 8-task session. Everything below is traced from real code (file
paths cited throughout) in the `app-studio` monorepo
(`C:\BizFirstGO_FI_AI\BizFirstAiStudio\src\app-studio`) unless marked **OPEN QUESTION**. See
`design-and-plan.md` for what was actually built this session and `lessons/README.md` for the
concrete, surprising findings along the way.

## 1. The widget-handler-registry pattern — no switch statements, ever

Every widget type is its own npm package (`packages/widget-handlers-{type}-widget`) implementing
`IWidgetHandler` (`packages/widget-handlers-core/src/IWidgetHandler.ts`):

```ts
export interface IWidgetHandler {
  readonly widgetType: string;
  load(widget: WidgetRecord): Promise<void>;
  render(ctx: WidgetRenderContext): WidgetRenderResult;   // pure data resolution, no DOM/React
  unload(widgetId: string): Promise<void>;
  readonly Component?: ComponentType<WidgetHandlerRenderProps>;  // mounts render()'s result
}
```

A runtime `WidgetRegistry` (`packages/widget-handlers-generic`) is populated once via
`registry.register(widgetType, new XxxWidgetHandler())` calls in
`useLivePreviewEngine.ts`/`app-player`'s own `App.tsx`, then looked up purely by string key
(`registry.get(widgetType)`) everywhere a widget needs to render — **confirmed zero switch
statements or `if (type === ...)` chains anywhere in the render path.** Adding a new widget type
means adding a new package + one registration line, never touching existing handler code. This is
the single most load-bearing architectural fact for any future automation targeting App Studio: a
new capability (a new embeddable content type) is additive by construction, not a modification.

**The real, current widget types** (verified via `WidgetRecord.ts`'s `WidgetType` union and the 9
real `packages/widget-handlers-*-widget` directories that exist as of this session — do not trust
this list without re-grepping if picking this up later, it grows):
`form`, `content`, `workflow-template`, `workflow-template-category`, `chat-panel`,
`page-navigation`, `hil-inbox`, `signin`, `notifications`, `site-branding`. Full per-type reference:
`rag/v1/widgetTypes/*.md` in this project folder.

## 2. The App/AppPage/AppSection/AppWidget data model

```
App (AIExt_Apps)
 ├─ Configuration (JSON string) → { meta: AppMeta, layout: AppLayout }
 │    AppLayout { type: AppLayoutType, appSections: AppSection[] }
 ├─ AppPage[] (AIExt_AppPages) — named, menu-able, routable pages
 └─ AppWidget[] (AIExt_AppWidgets) — placements, each pointing at a shared Widget + a section
```

**`AppSection` is recursive — `appSections?: AppSection[]` on itself** (`AppSection.ts`). This is
real, typed, and used for pure layout grouping (e.g. a `"header"` section containing
`"header-brand"`/`"header-nav"` children, each independently widget-bearing and further-nestable).
**This was a genuine, non-obvious gap for most of this session**: the Designer's left Sections list
and all three section-mutating store actions (`moveSection`/`removeSection`/`renameSection`)
originally only ever operated on the top-level flat `appSections` array — nested sections and their
widgets were completely invisible/unmanageable in the Designer, even though the real render engine
(`AppPlayer.tsx`) already handled nesting correctly for output. Fixed this session via a shared
recursive tree-helper module, `packages/app-handlers-core/src/utils/sectionTree.ts`
(`findSectionEntry`/`updateSectionTree`/`swapSectionInTree`/`collectSectionNames`) — the one place
all section-tree logic now lives. **If you are about to write ANY code that walks
`AppSection.appSections`, use these helpers, don't re-hand-roll flat-array logic** — that mistake
already cost real Designer functionality once.

`AppSection` other fields worth knowing: `region?: AppSectionRegion` (`'header'|'left'|'right'|
'main'|'footer'` — additive; omitted means the legacy flat/flex container, set on even one section
switches the renderer to a named-region CSS Grid), `widgetLayout?: AppSectionWidgetLayout`
(per-section flex direction/gap/align, independent of every other section), `isPrimaryContentSection`
(marks which section holds page-scoped content — see §4), `sectionContainer`/`sectionBackground`
(Level 2 style slots, see §3).

`AppPageRecord` (`AppPageRecord.ts`): `pageType: 'Physical'|'Virtual'`, `pageCategory:
'DataPage'|'ContentPage'`, `parentPageID?` (self-referencing FK, drives an expandable page-tree
widget), `menuTargetType: 'Page'|'ExternalUrl'|'Anchor'|null` (a menu item's click is polymorphic —
matches Wix/Webflow/Squarespace's own menu-item type selector), and (new this session)
`seo?: AppPageSeo` — see `rag/v1/00-overview.md` for the full field table.

`AppWidgetRecord` (`WidgetRecord.ts`): the placement join row — `sectionName`, `displayOrder`,
`appPageID?: number|null` (`null` = shared/default content; set only for widgets inside the
`isPrimaryContentSection` section, per-page scoping — see §4), `configuration`, and three levels of
style override (`widgetStyle`/`widgetCss` legacy flat fields, `styleConfiguration?: WidgetStyleConfig`
current structured one — see §3).

## 3. The structured Style Builder system — raw CSS is deliberately NOT the primary mechanism

Three levels, one shared value shape (`packages/app-handlers-core/src/types/StyleSlot.ts`):

```ts
interface StyleSlotValue { css?: string; style?: StyleProperties; }
// Level 1: SiteStyleConfig  { siteContainer?, siteBackground? }        — App.styleConfiguration
// Level 2: AppSection       { sectionContainer?, sectionBackground? }  — rides in layout JSON, no own column
// Level 3: WidgetStyleConfig (10 named sub-regions, only widgetContainer/widgetBackground wired
//          into the renderer today) — AppWidget.styleConfiguration
```

`style: StyleProperties` is a ~90-property, camelCase, structured object (colors, box model,
typography, flex/grid, position/z-index — full field table in `rag/v1/02-style-properties.md`),
edited via the shared `StyleBuilderPanel` component
(`atlas-forms/packages/designer-components-react/src/components/StyleBuilderPanel`) — reused
as-is from Atlas Forms, not reimplemented for App Studio. At render time (`app-handlers-generic`'s
`useStyleSlot.ts`) it's cast straight to `React.CSSProperties`, zero transformation.

`css: string` is a deliberate, narrow escape hatch — free-text CSS declarations, NOT selectors/rules,
sanitized (`cssInjector.ts`'s `sanitizeScopedCss`) and scoped to a deterministic hashed className so
identical text across instances shares one injected `<style>` rule. **The sanitizer actively rejects
`{`, `}`, `@`** — this is a hard security boundary (the field is free text an app-builder-level user
enters, spliced into a real `<style>` tag served to every end user), and it means **responsive
breakpoints cannot be bolted onto this field even as a stopgap** (no `@media`, no `@container`) — a
genuinely separate, structured mechanism would be needed (designed, not yet built — see
`design-and-plan.md`'s Task 6).

**Why structured-first matters for automation**: an agent generating App Studio styling should
almost always emit the `style` object (typed, enumerable properties, no CSS syntax to get wrong),
reaching for `css` only for the rare declaration `StyleProperties` genuinely has no field for.

## 4. Page-first content resolution — a Page is not a second routing system

`AppPageRecord`'s own doc comment is explicit: a Page "purely resolves TO a `navigate()` call, it is
not a second routing system." The real routing/content resolution lives in `RouteResolverService`
(URL ↔ AppCode/PageSlug) and `AppEngine.navigate()`/`previewNavigate()`. Widgets inside the section
flagged `isPrimaryContentSection` carry an optional `appPageID` — `null` means shared/default content
(what every pre-existing AppWidget implicitly is), a real ID scopes that widget to exactly one Page.
This is how a multi-page site (qoboto has 1 page; a real multi-page app would have several) shows
different primary content per page while sharing the same header/nav/footer sections.

## 5. Auth: `SsoAuthGate` + handoff-code SSO, and the `@passport/fake-auth` dev bypass

Every real App Studio / doc-app frontend uses the same pattern (`@bizfirst/common-auth-react`):
`<SsoAuthGate appName="..." apiBaseUrl={...}>{() => <RealApp/>}</SsoAuthGate>` — children only mount
once authenticated; an unauthenticated visit redirects to the central login app (`localhost:8001`)
with a `returnUrl`, which redirects back with a one-time `?code=` handoff token, exchanged for a
real bearer token via `useSsoHandoffSession`'s `authClient.redeemHandoffCode(code)`. **If a user
already has a valid session at the login-app level, this whole round-trip completes silently — no
password re-entry** — this is what makes cross-app "already logged in" navigation feel instant.

**`@passport/fake-auth`** (`passport/packages/@passport/fake-auth`) is a real, sanctioned,
**DEV-ONLY** bypass: seed `localStorage['auth-store']` directly with a `FAKE_TOKEN_PREFIX`-prefixed
token + a `FAKE_USER` object shaped exactly like `@passport/store`'s real `User` type, before the
app's first script runs. `useSsoHandoffSession`/every app's own session-check recognizes the prefix
and skips real `verifyToken()` for it entirely. Exact recipe (confirmed working this session against
`digital-assets-library`):

```js
localStorage.setItem('auth-store', JSON.stringify({
  state: {
    user: { id: '1', email: 'fake.user@bizfirstai.com', name: 'Fake User', roles: ['IsDeveloper'], permissions: [] },
    token: 'fake-debug-token-' + Date.now(),
    refreshToken: null, expiresAt: null, isAuthenticated: true,
  },
  version: 0,
}));
```

**Hard limit, non-negotiable**: this defeats the FRONTEND login gate only — it can never pass real
backend authorization (a fake token 401s on any real authenticated API call), and it must never be
left seeded on an origin you're not actively using for isolated frontend-only testing (it silently
made one origin show an empty "no apps" gallery this session because the real backend correctly
rejected every API call under that identity — always clear it when done, and never seed it as a
substitute for asking a human to log in when a real backend round-trip is actually needed). **A
coding agent must never enter a real password into any login field, under any circumstance, even
with explicit user permission to do so** — this is a hard operating rule for any future automation
in this codebase, not App-Studio-specific.

## 6. Cross-workspace package consumption — the established recipe

`app-studio` and `doc-app` (digital-assets-library, document-manager, knowledge-app) are **separate
pnpm workspaces**, each with its own `pnpm-workspace.yaml`, under one umbrella top-level workspace
(`BizFirstAiStudio/pnpm-workspace.yaml`, `packages: ['packages/*','src/**']` — broad enough to
technically include both, but each app's own Vite dev server only resolves through its OWN nested
`pnpm-workspace.yaml` + `vite.config.ts` aliases, not the umbrella one). `app-studio`'s own
`pnpm-workspace.yaml` already has three real precedents for reaching into a sibling workspace:
`atlas-forms`, `bizfirst-common`, `expressions` — each via an explicit relative-path member entry
(`- "../atlas-forms/packages/xyz"`) **plus** a matching alias in the consuming app's
`vite.config.ts` (`'@atlas-forms/xyz': path.resolve(__dirname, '../../../atlas-forms/packages/xyz/src')`).
**This is the exact recipe to follow for `app-studio` to reach `doc-app` packages** (e.g. reusing
digital-assets-library's `AssetGrid` picker from a content-widget's Insert Image button) — add the
relative-path member + the Vite alias, don't invent a new mechanism. Not yet wired as of this
session (Task 7, `design-and-plan.md`, is queued behind it).

## 7. DB standards enforced project-wide (CLAUDE.md)

Every new/altered table this session followed the same required shape: `Deleted`, `Archived`,
`LastModifiedOn`/`LastModifiedBy`, `CreatedOn`/`CreatedBy` (INT, never a string), `SourceAppID`,
`ClientAccountID`, `AppDomainID`, `DataDomainID`, `DataSegmentID`, `TenantID`, `ResID`
(UNIQUEIDENTIFIER, `DEFAULT (newid())`) — `DATETIME`, never `DATETIME2` — named `PK_`/`DF_`/`FK_`/
`IX_`/`UQ_`/`CK_` constraints — "ID" uppercase everywhere in code, never "Id". See
`lessons/README.md` for two real times this session where the LIVE database drifted from the
declarative SSDT source of truth (`dbo\Tables\*.sql`) that's supposed to encode these standards, and
why checking both directions of drift matters.

## 8. Open questions — explicitly unresolved, not guessed at

1. **Free-form/absolute widget positioning** does not exist in the current data model — `AppSection`
   has `widgetLayout` (flex config, section-wide) and `AppWidgetRecord` has only `displayOrder`
   (ordinal), no `x`/`y`/`top`/`left`. Designed (not built) this session — see `design-and-plan.md`
   Task 3 — as an additive `AppSection.layoutMode?: 'flow'|'canvas'` opt-in, riding coordinate data
   inside the existing `styleConfiguration` JSON (no new DB columns needed) rather than replacing the
   flow model.
2. **Responsive/per-breakpoint styling** does not exist yet either — `StyleSlotValue.style` is a
   single flat object. Designed (not built) as a structurally-discriminated `ResponsiveStyleValue
   {base, tablet?, mobile?}` shape using **CSS Container Queries, not `@media`** (a real, non-obvious
   design call — `@media` would report the Designer's own panel width incorrectly; `@container`
   unifies Designer-preview and real-runtime behavior) — see `design-and-plan.md` Task 6.
3. **Which concrete service backs `IDocumentStorageProvider`** for doc-app's document/asset storage
   was found to have NO public-read/ACL capability at all in `Platform.StorageServers`/
   `PresignService` (grepped, zero hits) — an interim `LocalDocumentStorageProvider` (local disk) was
   built this session specifically because of this gap, flagged as swappable, not a permanent answer.
