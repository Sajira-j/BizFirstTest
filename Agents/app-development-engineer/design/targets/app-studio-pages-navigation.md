# Target: App Studio — Pages (Physical/Virtual) as a real database entity + navigation menu

## Problem statement

Binoy, verbatim across three messages: (1) "An app like this should have pages. When user clicks
on a page it will change the content section and display that page. it can be a datapage or
content page. pages are listed as menu items." (2) "a page can be a virtual or a physical design -
but we need the ability to create pages. and a page may or may not have a menu item. a pages must
also be a database table with pagetype and pagecategory as well." (3) "do a comprhensive design.
do a detailed research and pull down industry standards and do a proper design."

Confirmed by reading the actual code: App Studio has no Page concept today. An app is one flat
layout of sections. But — and this changes the whole shape of this target — **App Studio already
has a complete, real, working SPA router underneath**, just nothing that creates named, persisted,
menu-able "pages" on top of it. This is not a green-field feature; it's a UI + one new table on
top of an engine that already works.

## Industry research — synthesis (full findings below; this is the design rationale)

Researched WordPress, Webflow, SharePoint modern pages, OutSystems, Power Apps, Retool, and
Next.js/Remix routing conventions. One pattern recurs across every single one of them, split along
exactly the axis Binoy described as "virtual vs physical":

- **Physical = a fixed, uniquely-authored composition.** Webflow's *Static Page*, WordPress's
  ordinary *Page*, Next.js's *static route* (`/about/page.tsx`), SharePoint's manually-built
  *Article page*. One design, always the same content, authored once.
- **Virtual = one composition, many records, parameterized by URL/data.** Webflow's *CMS
  Collection Page* (one template, Webflow generates one page per collection item automatically),
  Next.js's *dynamic route* (`/blog/[slug]/page.tsx` — one file serves every post), Power Apps'
  *model-driven Form/View* (auto-rendered from a table's schema, not hand-built per record). This
  is the closest industry match to Binoy's "data page."
- **Menu membership is universally a separate, optional concern from the page's existence.**
  WordPress models a nav menu item as its own row (a `nav_menu_item` post referencing the target),
  decoupled from the page itself. Retool confirms pages/apps remain independently reachable by
  direct URL regardless of navigation placement, and menu visibility is commonly permission-gated
  separately from the page's own access. This confirms "a page may or may not have a menu item" is
  the industry default, not an unusual ask — build it that way with confidence, no need to
  re-confirm with Binoy.

Design conclusion drawn from this: a **Physical page** targets one fixed widget (no record
parameter). A **Virtual page** targets one widget but is parameterized by `entityType`/`dataID` —
the SAME page definition renders different content depending on which record's ID is in the route,
exactly like a Webflow Collection Page or a Next.js `[slug]` route. Both cases already have a
real, working resolution mechanism in this codebase — see below, don't build a second one.

## Part 1 — this is NOT green-field: the routing engine already exists and works

Confirmed by reading `AppEngine.ts`, `RouteResolverService.ts`, `BrowserHistoryService.ts`,
`AppNavigateParams.ts` in full. This is real, wired, working code today — not dead plumbing:

- **`AppEngine.navigate(params: AppNavigateParams)`** is a real public method. `AppNavigateParams`
  is `{widgetID?, formID?, entityType?, dataID?, sectionName?, replace?}`. It resolves `params` via
  `RouteResolverService.resolve()`, applies the result to engine state (`applyRouteTarget` — sets
  `app.route.widgetID/formID/entityType/dataID`, sets `renderedWidgets.set(sectionName,
  appWidgetID)` — the exact map `AppPlayer.tsx`'s `getWidgetsForSection` already reads to show only
  the targeted widget), and pushes a **real, bookmarkable browser URL** via `BrowserHistoryService`
  (`window.history.pushState`/`replaceState`, real `popstate` handling already wired in the
  constructor, back/forward already works).
- **`RouteResolverService.resolve(params)`**: finds the section flagged `isPrimaryContentSection`,
  then picks a widget within it — by `widgetID` if given, by `formID` if given (matching a widget
  whose merged config has that `formID`), else the section's `isDefault` widget, else the first
  widget. Returns a `RouteTarget` (`{appWidgetID, sectionName, formID?, routeTemplate,
  routeSegments, routable}`).
- **This means "physical vs virtual" is already representable with zero new engine code**:
  - Physical page → `navigate({widgetID: N})` — always the same widget, no record parameter.
  - Virtual page → `navigate({widgetID: N, entityType: 'Order', dataID: recordID})` — same widget
    (e.g. a `form` widget in `view`/`edit` mode), but the `entityType`/`dataID` route segments
    (already flow into `TokenResolverService`'s `{{url.segment.X}}` token scope, confirmed in
    `AppEngine.executeAction`'s `resolveToken` call) parameterize what it shows. This is exactly
    the Webflow Collection Page / Next.js `[param]` pattern, already mechanically supported.
- **A `'navigate'` app-scope action is already registered** in `AppEngine.registerBuiltInActions()`
  — a widget's own click handler can already trigger real navigation today, this target doesn't
  need to invent that either.
- **The actual, confirmed gap**: nothing creates a NAMED, PERSISTED "this is a page, here's its
  title/slug/menu placement" record — today `navigate()` only takes raw `widgetID`/`formID`/
  `entityType`/`dataID`, which a menu item would have to hardcode per-button. And nothing renders a
  menu at all (grepped `app-handlers-generic` + both apps for any component reading `showInNav`/
  `navPosition` — zero results).

## Part 2 — the real, current database/backend shape (read before designing the new table)

Confirmed by reading the actual EF entities and controller, not guessed from CLAUDE.md's abstract
rule:

- `App`, `AppWidget`, `Widget` (`BizFirstPayrollV3\src\mvc-server\Ai\AIExtension\
  BizFirst.Ai.AIExtension.Domain\Entities\`) all inherit `BaseEntity` — table names are the entity
  name prefixed `AIExt_` (`AIExt_Apps`, `AIExt_AppWidgets`, `AIExt_Widgets`). `BaseEntity`'s own
  source wasn't found in this repo (likely a referenced shared framework assembly) — treat
  CLAUDE.md rule 3's column list as authoritative for what it provides
  (Deleted/Archived/LastModifiedOn/By/CreatedOn/By/SourceAppID/ClientAccountID/AppDomainID/
  DataDomainID/DataSegmentID/TenantID/ResID) and confirm the exact base-class source at
  implementation time rather than re-guessing.
- **`WidgetType` and `NavPosition`'s real, confirmed convention**: both are plain
  `[StringLength(N)] string` columns on the entity, each with a code comment stating "enforced by
  DB CHECK constraint and re-validated at the service layer." **Not** a separate lookup table, not
  an int enum. `PageType` and `PageCategory` must follow this exact, already-proven convention —
  `[StringLength(50)] string` + a DB `CHECK` constraint + service-layer re-validation, matching
  `Widget.WidgetType`/`AppWidget.NavPosition` byte-for-byte, not inventing a new persistence style
  for this one table.
- **`AppWidget`'s real, current columns relevant here**: `ShowInNav` (bool), `NavPosition`
  (nullable string, "top"/"side", CHECK-constrained), `Routable` (bool, doc comment: "consumed by
  the App Studio routing system — a routable widget gets an automatic route"). These already exist,
  already round-trip through `BaseAppStudioAppWidgetController`'s `Update` action (a nullable-DTO
  partial-PATCH-via-PUT pattern — `UpdateAppWidgetRequestDto` has all-optional fields, controller
  applies only what's non-null). **Design decision, resolved**: keep these fields on `AppWidget` as
  a widget-level default/fallback (a widget can still be independently routable without a formal
  Page), but a `Page` is the new, richer, nameable/menu-configurable wrapper — a Page references a
  widget (physical) or a widget+entityType (virtual), and Page-level `ShowInMenu`/`MenuLabel`/
  `MenuPosition`/`DisplayOrder` are what the new nav menu actually reads. Do not deprecate
  `AppWidget.ShowInNav`/`NavPosition`/`Routable` — they stay as the widget's own routability flag
  (whether a route CAN target it at all); the Page entity is what makes a route a real, named,
  creatable, menu-placeable thing. This mirrors WordPress's own separation (a post has
  `post_status`; a menu item is still its own separate row referencing it).
- **Controller pattern to mirror exactly** (`BaseAppStudioAppWidgetController.cs`): abstract base
  controller class + `IAppStudioXService` interface, `[ApiLimit]` + `[AuthorizeRegularUserAttribute]`
  (read) / `[AuthorizeTenantAdminAttribute]` (write) attributes, `GetByIdWebRequest`/`IDInfo`
  request-envelope conventions, soft-delete via `SoftDeleteAsync`, nullable-field partial-update
  DTOs. A new `BaseAppStudioPageController` must follow this pattern precisely, not invent a new
  controller shape.

## Part 3 — the `AppPage` entity (design, resolved)

New table `AIExt_AppPages` (matching the `AIExt_` prefix convention), new EF entity `AppPage :
BaseEntity` in the same `Entities` folder as `App`/`AppWidget`/`Widget`:

```csharp
public class AppPage : BaseEntity
{
    [Key]
    public int AppPageID { get; set; }

    [Required]
    public int AppID { get; set; }                    // FK -> AIExt_Apps.AppID

    [Required, StringLength(200)]
    public string Name { get; set; } = string.Empty;   // internal name, matches Widget.Name convention

    [Required, StringLength(200)]
    public string Title { get; set; } = string.Empty;  // display title shown on the page itself

    [Required, StringLength(200)]
    public string Slug { get; set; } = string.Empty;   // URL segment, unique per AppID (DB unique index)

    /// PageType: "Physical" | "Virtual" — CHECK-constrained, same convention as
    /// Widget.WidgetType / AppWidget.NavPosition (see Part 2). Physical = fixed widget target,
    /// no record parameter. Virtual = same widget, parameterized by EntityType+a runtime DataID
    /// (Webflow Collection Page / Next.js [param]-route analog — see industry synthesis above).
    [Required, StringLength(50)]
    public string PageType { get; set; } = "Physical";

    /// PageCategory: "DataPage" | "ContentPage" — CHECK-constrained, same convention. Matches
    /// Binoy's own framing ("it can be a datapage or content page") and this codebase's existing
    /// Widget.WidgetType split (form/list-typed widgets = data-driven; content-typed = static).
    [Required, StringLength(50)]
    public string PageCategory { get; set; } = "ContentPage";

    [Required]
    public int AppWidgetID { get; set; }               // FK -> AIExt_AppWidgets.AppWidgetID — the widget this page targets

    /// Only meaningful when PageType = "Virtual". Matches AppNavigateParams.entityType exactly —
    /// this is not a new routing concept, it's the persisted name for a parameter the engine
    /// already accepts at navigate() time.
    [StringLength(100)]
    public string? EntityType { get; set; }

    /// "May or may not have a menu item" (Binoy, explicit requirement) — modeled as nullable/
    /// optional fields directly on AppPage, not a separate MenuItem entity. Justification: a menu
    /// item here has no independent identity or behavior beyond "show this page in the menu, with
    /// this label, at this position" — a separate table would be an unjustified join for a
    /// 1:0-or-1 relationship with no fields of its own beyond what's listed here (this repo's own
    /// "no premature abstraction" rule). WordPress's separate nav_menu_item row exists because a
    /// menu item there can target things that are NOT pages (external URLs, categories); this
    /// codebase's menu only ever targets one of ITS OWN Pages, so the WordPress split doesn't
    /// apply here.
    public bool ShowInMenu { get; set; } = false;

    [StringLength(200)]
    public string? MenuLabel { get; set; }              // shown in the menu; may differ from Title

    [StringLength(50)]
    public string? MenuPosition { get; set; }           // "top" | "side" — same CHECK-constraint convention as AppWidget.NavPosition

    public int DisplayOrder { get; set; } = 0;          // menu ordering, matches Widget.DisplayOrder convention

    public bool IsDefault { get; set; } = false;        // the page an app loads when opened with no route — at most one true per AppID (service-layer + filtered unique index, matching AppWidget.IsDefault's own documented enforcement)

    public bool IsActive { get; set; } = true;
}
```

Standard `BaseEntity` columns (Deleted/Archived/LastModifiedOn/By/CreatedOn/By/SourceAppID/
ClientAccountID/AppDomainID/DataDomainID/DataSegmentID/TenantID/ResID) come from the base class
like every sibling entity — do not redeclare them.

**DDL note**: `DATETIME` (not `DATETIME2`), named `DF_`/`PK_` constraints, a real `CHECK`
constraint on `PageType`/`PageCategory`/`MenuPosition` (values above) — confirm the exact
constraint-naming convention against whatever migration/DDL script created `AIExt_AppWidgets`
(not located during this research pass; find and mirror it exactly at implementation time rather
than inventing a naming scheme).

## Part 4 — how Physical vs Virtual resolve at render time (reconciled with the real engine, no second router)

A page navigation is just a call to the *already-real* `AppEngine.navigate()`:

- **Physical**: `engine.navigate({ widgetID: page.appWidgetID })`.
- **Virtual**: `engine.navigate({ widgetID: page.appWidgetID, entityType: page.entityType, dataID: <runtime record ID, e.g. from a clicked list row> })` — the SAME page definition, different `dataID` per invocation. A virtual page's menu entry (if `ShowInMenu`) would typically link to a *list* of records (rendered by some other widget/page) whose row-click action supplies the `dataID` — the menu item itself just gets you to the page in a default/unparameterized state; the FRONTEND components (a list widget's row-click `ActionBinding`, already a real, existing mechanism) supply the record parameter. **Do not build page-specific new click plumbing** — reuse the existing `ActionBinding`/`executeAction`/`navigate`-action system exactly as `RouteResolverService`'s own `formID`-matching already assumes.

No second routing/resolution system. `RouteResolverService` doesn't need to know about `AppPage`
at all — `AppPage` is purely a design-time/menu-time concept that resolves TO a call into the
existing engine API. Confirm this reconciliation against `AppEngine.ts`'s current state at
implementation time — Part 1's snapshot is accurate as of 2026-08-26, but check for drift.

## Part 5 — phase breakdown

### Phase 0 — verify, don't re-derive
- Re-confirm every file/entity snapshot in Parts 1–2 against current code — this session moves
  fast, code may have shifted. Two other target specs (`app-studio-section-layout-model` — was
  mid-build as of this research, `app-studio-project-app-unification` — not yet started) touch
  `AppSection.ts`/`AppPlayer.tsx`/`CanvasPanel.tsx`/`AppRecord`; check their actual landed state
  (not just their spec files) before starting, since this target also touches
  `AppSection.isPrimaryContentSection` and the designer canvas.
- Locate `BaseEntity`'s real source and the actual DDL/migration script that created
  `AIExt_AppWidgets`, to mirror exact constraint naming — not found during this research pass.

### Phase 1 — backend: entity, migration, controller
1. `AppPage` EF entity (Part 3), DbContext registration mirroring how `App`/`AppWidget`/`Widget`
   are registered in `AIExtensionDbContext.cs`.
2. Migration/DDL creating `AIExt_AppPages` — standard columns + the columns in Part 3, `CHECK`
   constraints for `PageType`/`PageCategory`/`MenuPosition`, a unique index on `(AppID, Slug)`, a
   filtered unique index enforcing at-most-one `IsDefault=true` per `AppID` (mirror whatever
   mechanism enforces `AppWidget.IsDefault`'s equivalent constraint — confirmed to exist per that
   entity's own doc comment, find and reuse the same technique).
3. `BaseAppStudioPageController` mirroring `BaseAppStudioAppWidgetController.cs`'s exact shape:
   `GET /api/v1/app-studio/apps/{appId}/pages` (list), `POST` (create), `PUT /{id}` (partial
   update via nullable DTO), `DELETE /{id}` (soft delete), plus a `reorder` action mirroring
   `set-default`'s shape for `DisplayOrder` changes across multiple pages at once.

### Phase 2 — frontend API client + types
1. `AppPageDto` in `app-studio-api-client-js/src/types.ts`, matching `AppWidgetDto`'s existing
   style.
2. `createAppPagesClient`/`AppPagesApiClient` in a new `pages.client.ts`, mirroring
   `app-widgets.client.ts`'s exact shape (`list`/`create`/`update`/`delete`).

### Phase 3 — designer UI: create/manage pages
1. A Pages management screen/panel in `app-studio-designer-components-react` — list pages for the
   current app, create (Title/Slug/PageType/PageCategory/target AppWidget picker/EntityType-if-
   Virtual/ShowInMenu+MenuLabel+MenuPosition), edit, delete, reorder. Reuse the existing
   `FormLookupField`-style lookup pattern (already used for `formID`/`executionTemplateID` this
   session) for picking the target `AppWidgetID` rather than a raw number input.
2. A way to mark a section `isPrimaryContentSection` (referenced in Part 1 as a prerequisite for
   `RouteResolverService.findContentPaneSection()` to find anything) — likely a single toggle on
   `SectionEditor.tsx`, validated to at most one per layout (match `findContentPaneSection`'s own
   current behavior with zero/multiple flagged sections — don't assume, verify).

### Phase 4 — the real end-user nav menu
1. A new `AppNavMenu` component (name TBD) in `app-handlers-generic` or a sibling, rendered by
   `AppPlayer.tsx`: reads all `AppPage`s for the app where `ShowInMenu` is true, grouped/sorted by
   `MenuPosition`/`DisplayOrder`, renders as real clickable items, `onClick` calls
   `engine.navigate({widgetID: page.appWidgetID, entityType: page.entityType})` (Part 4's Physical/
   Virtual resolution).
2. Registered in both `app-studio-designer`'s live-preview canvas AND `app-player`'s standalone
   runtime — the exact same registration-parity risk already flagged elsewhere in this session's
   QA log; keep both in sync deliberately.

## Constraints

- No auto-commits/pushes.
- `ID` uppercase in every new identifier (`AppPageID`, `AppWidgetID`, etc.) — this is a brand-new
  table/entity, not an existing file with a pre-existing lowercase-`d` convention to preserve, so
  CLAUDE.md's naming rule applies with no exception here.
- Additive-only on the frontend side — `AppWidget.ShowInNav`/`NavPosition`/`Routable` are
  unchanged; an app with zero `AppPage` rows must render exactly as it does today.
- No premature abstraction — one `AppPage` entity, no generic "menu item" or "route" abstraction
  beyond what's specified above; the routing engine itself already exists, don't rebuild it.
- `pnpm typecheck`/`pnpm build` clean for every touched frontend package; a real backend build
  (`dotnet build`) clean for the touched C# projects.
- Dated `DevelopmentHistoryLog.md` entries in every touched package/project.
- Live-verify: create a real Page in a real app, confirm it appears in the menu (if `ShowInMenu`),
  confirm clicking it swaps the content section to the target widget, confirm a Virtual page
  correctly parameterizes by a real `dataID` from a list-widget row click, in both the designer's
  live-preview canvas and `app-player`'s standalone runtime.
