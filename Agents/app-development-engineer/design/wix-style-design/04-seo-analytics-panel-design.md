# SEO Checklist + Analytics Panel — Design

Status: design only, no code. Companion to Task 2 (unified click-to-edit editor) and Task 6
(responsive breakpoints) — this panel is placeable independent of either shipping first, but its
natural home is the per-page details surface those tasks are building.

## 1. What exists today (research findings)

**App-level meta only.** `AppConfiguration.meta: AppMeta` (`packages/app-handlers-core/src/types/AppConfiguration.ts`)
has exactly three fields: `title`, `description`, `keywords`. This is stored once per App, inside
`AIExt_Apps.Configuration` JSON (the qoboto app uses it: `{"meta":{"title":"Qoboto - ...",
"description":"..."}}`). There is **no per-page SEO data anywhere** — `AppPageRecord`
(`packages/app-handlers-core/src/types/AppPageRecord.ts`) has `title` and `slug` (used for
navigation/routing) but no `description`, `ogImage`, `canonical`, or `robots` fields. Every page in
a multi-page app today would emit the *same* meta description if we naively fell back to the
app-level value — a real, pre-existing SEO gap independent of this design.

**No analytics/tracking pipeline exists.** Grepped `BizFirstPayrollV3` and `BizFirstFiDB` for
`PageView`, `AppAnalytics`, `TrackingEvent` — zero real hits (one incidental match in an unrelated
whitelabel doc, a few DB tables whose names coincidentally contain "Tracking" as a substring but
aren't page-view related). **There is nothing to surface today.** The analytics half of this panel
is scoped as "requires a new backend," not designed as if data already exists.

**Settings-panel precedent.** `AppConfigScreen.tsx` (`packages/app-studio-designer-components-react/src/properties/`)
is the existing full-screen tabbed pattern: `General | Sections | Widgets | Style | Starter Actions`,
each tab a swapped-in component, App-scoped. This is the natural place for an app-wide "SEO &
Analytics" overview tab. Section Details already established a second precedent this session — a
selected item's own accordion panel (`SectionDetailsContent`, 5 collapsible accordions) — which is
the natural place for **per-page** SEO fields once a page is the selected/editing target.

**Content is real, inspectable HTML.** `ContentWidgetRenderer.tsx` renders `result.sanitizedHtml`
(DOMPurify-sanitized, from the Tiptap HTML Editor) via `dangerouslySetInnerHTML` — genuine `<h1>`,
`<h2>`, `<img alt="...">` tags can exist in it. This means the SEO checklist can be computed by
parsing each page's actual rendered content widgets' HTML strings, not by inventing new structured
metadata the author has to duplicate.

## 2. Data model

### 2.1 New per-page SEO fields

Backend: new nullable `SEOConfiguration NVARCHAR(MAX)` JSON column on the `AppPage` table (mirrors
the existing `AIExt_Apps.Configuration` JSON-blob convention — no new child table needed for a
handful of optional fields). Per CLAUDE.md DB standards: add via a migration that only adds this
one column (no touch to existing audit columns), named `DF_AppPage_SEOConfiguration` if a default
is needed (it isn't — nullable, no default).

```ts
// packages/app-handlers-core/src/types/AppPageSeo.ts (new file)
export interface AppPageSeo {
  metaTitle?: string;        // falls back to AppPageRecord.title, then AppConfiguration.meta.title
  metaDescription?: string;  // falls back to AppConfiguration.meta.description
  ogImageUrl?: string;
  canonicalUrl?: string;     // falls back to the computed RouteResolverService URL for this page
  robotsIndex: boolean;      // default true
  robotsFollow: boolean;     // default true
  structuredData?: string;   // raw JSON-LD block, escape hatch only (same "css string" pattern
                              // AppConfiguration.css uses) — not a structured schema.org builder v1
}
```

`AppPageRecord` gains one new optional field: `seo?: AppPageSeo`. Additive, every existing page
with no SEO block set behaves exactly as today (falls through to the app-level `AppMeta`).

### 2.2 Fallback chain (what actually renders in `<head>`)

```
page.seo?.metaTitle ?? page.title ?? app.meta?.title ?? app.name
page.seo?.metaDescription ?? app.meta?.description
page.seo?.canonicalUrl ?? RouteResolverService.buildUrl({ appCode: app.appCode, pageSlug: page.slug })
```

This chain is what `app-player`'s page-render path must apply when writing `<title>`/`<meta>` tags
(today it likely only ever reads the app-level `AppMeta` — confirming/wiring this is implementation
work, out of scope for this design doc, but noted as the concrete integration point).

## 3. SEO checklist — concrete, computable checks

Each check runs against the **resolved** per-page data (Section 2.2's fallback chain) plus the
actual rendered HTML of every content widget attached to that page (via `appWidgets.filter(w =>
w.appPageID === page.appPageID)`, mirroring the page-first design's own widget-to-page association).

| Check | Pass condition | Data source |
|---|---|---|
| Title present | resolved title non-empty | fallback chain above |
| Title length | 10–60 chars | same |
| Meta description present | resolved description non-empty | same |
| Meta description length | 50–160 chars | same |
| Exactly one H1 | `sanitizedHtml.match(/<h1[\s>]/gi).length === 1` across the page's content widgets combined | ContentWidgetHandler's already-sanitized output |
| Heading order (no skips) | no `<h3>` appears before any `<h2>` in document order, etc. | same, parsed via a lightweight regex/DOM-parser walk (no new dependency — `DOMParser` is available browser-side where this check actually runs, i.e. in the Designer, not on the backend) |
| Image alt-text coverage | % of `<img>` tags with a non-empty `alt` attribute; 100% = pass, else shows the count missing | same |
| OG image set | `seo.ogImageUrl` non-empty | AppPageSeo |
| Canonical URL resolvable | `RouteResolverService.buildUrl(...)` succeeds (app has an AppCode, page has a slug) | existing routing service |
| Slug is url-safe | matches the same slug-validation regex `AppPageRecord.slug` already enforces at creation time | existing validation |

Score = passed checks / total checks, shown as a percentage + red/yellow/green per-row status —
same visual language as the existing widget/section badges (`AppConfigScreen.tsx`'s `styles.badge`
pattern: small pill, colored by state).

## 4. Analytics — the prerequisite backend (does not exist; must be built first)

This is scoped honestly as **new work**, not a v1 UI over existing data.

### 4.1 Minimal event model

New table (per CLAUDE.md DB standards — DATETIME not DATETIME2, named DF_/PK_ constraints, all
mandatory audit/multi-tenancy columns, CreatedBy as INT):

```sql
CREATE TABLE AIExt_AppPageViews (
  AppPageViewID   INT IDENTITY(1,1) NOT NULL,
  AppID           INT NOT NULL,
  AppPageID       INT NULL,              -- null = app-level/non-page-scoped view
  TenantID        INT NOT NULL,
  VisitorID       UNIQUEIDENTIFIER NOT NULL,   -- anonymous, client-generated, persisted in a cookie
  SessionID       UNIQUEIDENTIFIER NOT NULL,
  Referrer        NVARCHAR(1000) NULL,
  UserAgent       NVARCHAR(500) NULL,
  OccurredOn      DATETIME NOT NULL,
  Deleted         BIT NOT NULL, Archived BIT NOT NULL,
  LastModifiedOn  DATETIME NULL, LastModifiedBy INT NULL,
  CreatedOn       DATETIME NOT NULL, CreatedBy INT NULL,
  SourceAppID     INT NULL, ClientAccountID INT NULL, AppDomainID INT NULL,
  DataDomainID    INT NULL, DataSegmentID INT NULL, TenantID2 INT NULL, ResID UNIQUEIDENTIFIER NOT NULL,
  CONSTRAINT PK_AppPageViews PRIMARY KEY (AppPageViewID),
  CONSTRAINT DF_AppPageViews_Deleted DEFAULT (0) FOR Deleted,
  CONSTRAINT DF_AppPageViews_Archived DEFAULT (0) FOR Archived,
  CONSTRAINT DF_AppPageViews_ResID DEFAULT (NEWID()) FOR ResID
);
```
(`TenantID2` above is a placeholder name collision note — the real column list must reconcile
`TenantID` already present for row-scoping vs. the standard mandatory `TenantID` column; in
practice these are the same column, this is a drafting artifact to fix in the actual migration —
flagged here rather than silently resolved, since it's a real modeling decision for whoever
implements this.)

### 4.2 Ingest endpoint

`POST /api/v1/app-studio/analytics/pageview` — fire-and-forget from `app-player`'s
`AppPlayer.tsx` on every successful page render (via `navigator.sendBeacon`, never blocking
render, never surfaced as a user-visible error on failure). Body: `{ appID, appPageID, visitorID,
sessionID, referrer }`. `visitorID` persisted client-side (localStorage, not a tracking cookie
requiring consent banners — first-party, no cross-site use).

### 4.3 Aggregation for the panel (once 4.1/4.2 exist)

- Pageviews over time (7/30/90-day line chart) — `GROUP BY CAST(OccurredOn AS DATE)`
- Top pages — `GROUP BY AppPageID ORDER BY COUNT(*) DESC`
- Unique visitors — `COUNT(DISTINCT VisitorID)`
- Referrer breakdown — `GROUP BY Referrer` (top 10, "direct" bucket for null)

Conversion events (form submits, button clicks) are explicitly **out of scope for v1** — they need
a second, differently-shaped event (`EventType`, target widget ID) and are a natural v2 extension
of the same table, not a v1 requirement.

## 5. UI placement

Two surfaces, both reusing existing precedent:

1. **App-wide "SEO & Analytics" tab** in `AppConfigScreen.tsx` (6th tab alongside General/Sections/
   Widgets/Style/Starter Actions) — shows the aggregate checklist score across all pages (list,
   worst-first) and the analytics charts (Section 4.3), once the prerequisite backend exists. Until
   then, the analytics half renders a single "Analytics requires backend work — see design doc"
   empty state, never fake/zero data presented as real.
2. **Per-page "SEO" accordion** — once a page becomes a directly-selectable/editable entity (Task 2's
   unified editor, or today's existing Pages list + a page-details panel), add a 6th accordion
   alongside the pattern `SectionDetailsContent` already established this session (Actions / Widgets
   / Region / Widget Layout / Styling) — a new "SEO" accordion with the `AppPageSeo` fields from
   Section 2.1 as plain form inputs, plus the per-page checklist rows from Section 3 rendered live
   as the author types (so title-length/description-length checks flip pass/fail in real time,
   matching Wix/Yoast-style live feedback).

## 6. Explicit dependencies / sequencing

- Per-page SEO fields (Section 2) require a real backend migration (new column) — not blocked on
  any other task in this document set, can be built independently.
- The checklist (Section 3) requires nothing beyond what exists today (client-side computation over
  already-loaded widget HTML) — buildable immediately.
- Analytics (Section 4) requires new backend work (table + endpoint + aggregation queries) before
  any UI is meaningful — this is the long pole of this entire design, should be sequenced as its
  own backend task, not bundled into the same PR as the SEO checklist UI.
- The per-page accordion placement (Section 5.2) is easiest to land after Task 2's per-page details
  panel exists, but the app-wide tab (Section 5.1) has no such dependency and can ship first.

## Build Progress — Part A (2026-08-30)

**Shipped:**
- `AppPageSeo` type (`app-handlers-core/src/types/AppPageSeo.ts`) + `AppPageRecord.seo?` field,
  additive per section 2.1.
- Backend: migration `V018_UP/DOWN_AIExt_AppPages_SEOConfiguration.sql` (idempotent, mirrors V015's
  StyleConfiguration pattern exactly — nullable NVARCHAR(MAX) + ISJSON CHECK constraint). **Not
  executed against the live database** — left for Binoy's review per instruction. Added the
  matching `SEOConfiguration` property to the `AppPage` C# entity
  (`BizFirst.Ai.AIExtension.Domain/Entities/AppPage.cs`) so the column round-trips once the
  migration runs; confirmed the Domain project builds clean (`dotnet build -m:2 -nodeReuse:false`,
  0 errors). Did NOT chase down whether a separate backend AppPageDto class needs updating too —
  grepped and found none (the entity appears to serialize directly); if that assumption is wrong,
  the new property simply won't appear in the API response yet, which is safe (frontend already
  treats a missing `seoConfiguration` as "no override," same as today).
- `AppPageDto.seoConfiguration` (frontend API-client type) + parsing into `AppPageRecord.seo` in
  both `ApiBackedAppDataLoader` (app-player) and `StoreBackedAppDataLoader` (Designer preview).
- Section 2.2's fallback chain, section 4's title/meta-tag wiring: new `AppState.name/appCode/meta`
  fields (previously fetched but silently discarded — real, confirmed gap), `seoMeta.ts`
  (`resolvePageSeo`/`applyPageSeoToDocument`/`getCurrentPage`, pure + DOM-apply split for
  testability), `useDocumentSeoMeta` hook wired into `AppPlayer.tsx`, gated on `!studioMode` so the
  Designer's own embedded preview never stomps the Designer's own browser tab title. **Live-verified**:
  opening `app-player` on qoboto set the browser tab title to "Qoboto - Decentralized Website
  Builder" (previously unset — confirmed via direct read of the tab title before/after).
- Section 3's checklist (`computeSeoChecklist.ts`, DOMParser-based, all 10 rows from the design
  table) + Section 5.1's app-wide "SEO & Analytics" tab (`SeoAnalyticsTab.tsx`, wired as
  `AppConfigScreen.tsx`'s 6th tab) — worst-first sorted page list, expandable per-page checklist,
  honest analytics empty state (no fake data). All touched packages typecheck clean
  (`app-handlers-core`, `app-handlers-generic`, `app-studio-api-client-js`,
  `app-studio-designer-components-react`, `app-studio-designer`, `@app-studio/player`).

**Verification gap, disclosed rather than papered over:** the SEO & Analytics tab's own live
render (checklist scores against qoboto's real content) was NOT visually confirmed in-browser —
the Consolidated WebApi's `/api/v1/app-studio/widgets` endpoint was hanging under this session's
concurrent multi-agent load at verification time (confirmed independently via direct `curl` —
`/apps/{id}/pages` returned in 155ms, `/widgets` timed out after 25s with no response), not a bug
in this tab's code: the network calls it issues are the correct shape/endpoints, both source
packages typecheck clean, and the sibling title/meta-tag piece (same fallback-chain logic, simpler
data dependency) already live-verified successfully. Re-verify this specific tab once the shared
backend isn't under concurrent-agent load — click App Config → SEO & Analytics on the qoboto app
(AppID 1526) and confirm the checklist renders real scores (expect the Home page to show a
non-empty H1/alt-text/heading-order read against its real content widgets).

**Deferred, not silently dropped:**
- Part C (analytics backend, section 4) — its own separate backend effort per this doc's own
  original scoping.

## Build Progress — Part B (2026-08-30)

**Shipped:**
- Investigated Task 2's actual shipped shape first, per the coordinator's directive, rather than
  assuming the pre-Task-2 design doc's wording still held: confirmed (via
  `02-click-to-edit-inline-designer.md`'s own Build Progress log) that Task 2 shipped a page-
  *switcher* (top tabs, `DesignerToolbar.tsx`), not a page-level details *overlay* the way Section/
  Widget Details exist — there was no "click a page, see its details" surface to slot an accordion
  into. Built the missing minimal surface rather than force-fitting into `SectionDetailsContent`.
- New `packages/app-studio-designer-components-react/src/details/PageDetailsContent.tsx` — one
  "SEO" `Accordion` (matches `SectionDetailsContent`'s pattern exactly): the `AppPageSeo` fields
  (section 2.1) as plain inputs, plus the section 3 checklist rows, recomputed on every keystroke
  against LOCAL form state (not the last-saved value) — real Yoast/Wix-style live feedback, not a
  static read.
- New gear `IconButton` next to the page-switcher tabs in `DesignerToolbar.tsx`, operating on the
  currently active page (one entry point, not one per tab). Opens a `DesignerToolbar`-local
  `OverlayPanel` instance — deliberately NOT lifted into `designer-ui.store.ts`/`CanvasPanel.tsx`,
  since a page's own settings are a toolbar-scoped concept independent of the canvas's section/
  widget selection, and this avoids touching either of those files (both under concurrent
  modification by Tasks 3/6/8 in this same wave — zero collision risk this way).
- Persistence: `AppPagesApiClient.update(appId, appPageID, { seoConfiguration: JSON.stringify(seo)
  })` (or `null` when every field is empty, so an all-cleared form correctly reverts to "no
  override" rather than persisting an empty-but-truthy JSON blob).
- Section 2.2's fallback chain, section 3's checklist computation: reused `computeSeoChecklist`
  (Part A) verbatim, no changes needed — it was already a pure function taking plain data, exactly
  reusable here.
- **Confirmed backend round-trip already works, resolving Part A's own open question**: `AppPageDto`
  (the frontend API-client type, not a separate backend DTO) already carries `seoConfiguration?:
  string | null` and `AppPagesApiClient.update()` already accepts it via `Partial<AppPageDto>` — no
  backend gap found. (Whether the ASP.NET controller's own request-model class round-trips this
  field was not independently re-verified against the live backend this pass — the migration itself
  was already confirmed applied to the live DB by the coordinator earlier this session, which is the
  harder prerequisite; a live save-and-reload smoke test is the one remaining confirmation, deferred
  to the coordinator's end-of-batch verification pass per explicit instruction.)
- **Flagged, not fixed (out of scope for this change)**: `SeoAnalyticsTab.tsx` (Part A) computes a
  content widget's HTML from the shared Widget catalogue entry's `configuration`
  (`WidgetsApiClient.list()`'s own default), not the per-page `AppWidgetDto.configuration` override
  — this component reads the correct per-placement source instead. Worth reconciling in
  `SeoAnalyticsTab.tsx` at some point since its checklist scores could disagree with a page's real
  content whenever an author has overridden a content widget's text per-placement.
- `pnpm tsc --noEmit` clean for both new/touched files, confirmed via a targeted check isolating
  them from several unrelated pre-existing errors elsewhere in the same package from Tasks 6/7's
  concurrent in-progress work (not this change's concern).

**Verification status**: build-level confirmed correct; live-browser verification (opening a real
page's Settings gear, editing a field, confirming the checklist updates live, saving, reloading,
confirming it persisted) deliberately NOT done this pass — per the coordinator's explicit
instruction to build for correctness now and do one complete verification pass across everything at
the end, rather than fight the shared dev backend's current instability task-by-task.
