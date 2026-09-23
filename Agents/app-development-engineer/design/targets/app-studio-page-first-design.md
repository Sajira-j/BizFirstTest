1# Target: App Studio — Page-first design (Wix-style primary canvas) + page-scoped widgets + multi-target menu items

## Problem statement

Binoy, across several messages, live in the Designer:

1. "When I add a page, it asks me to pick up a widget. However, I should be able to redirect the
   user to any widget I please. However it expects me to add all widgets into content pane and
   select from there. That does not look like a good design."
2. Proposed two alternatives, then: "I think we need to think through and come up with a design
   here." Landed on a hybrid after discussion: a Page owns a set of widgets (not one), the primary
   section is an empty-by-default "outlet," and adding a widget while a page is being edited
   auto-tags it to that page — no separate assign-to-page step.
3. "What you think is a better choice. Any product like Wix, they lead with page design. Our
   design is anti usability pattern." — direct push to commit to page-first as the Designer's
   primary interaction model, not a secondary screen bolted onto section/layout editing.
4. "Write it up as a target spec... just copy the usability style of wix that would be more than
   enough."
5. "also remember, not all menu items load widgets. some menu items will open a url on a new
   window, some will just focus on a bookmark etc" / "keep various types of targets when designing
   menu item" — a menu item's target is broader than "navigate to a page."
6. "do you best efffort to research and fix all loop wholes and I want a world class design and
   ease of use for users."

This target supersedes part of `app-studio-pages-navigation.md` (built earlier the same day) —
that target's `AppPage`/`AppNavMenu`/routing-engine work is correct and stays; what changes is
(a) how a Page acquires its widgets, (b) the Designer's primary information architecture, and
(c) what a menu item can point at.

## Current state (confirmed 2026-08-26 evening, direct read — not the earlier spec's stale snapshot)

- `AppPage.cs` (`BizFirstPayrollV3\...\Entities\AppPage.cs`): `AppWidgetID` is `[Required] int` —
  a page targets exactly **one** widget today. `PageType`/`PageCategory`/`ShowInMenu`/
  `MenuLabel`/`MenuPosition`/`DisplayOrder`/`IsDefault`/`IsActive` all already exist and are
  correct as-is (see Part 3 below for what changes, additively).
- `AppWidget.cs`: `SectionName` is a plain required `[StringLength(200)] string` (not an FK/enum —
  matched against layout section names by value). Every existing FK on this entity (`AppID`,
  `WidgetID`) is non-nullable `int`. **No existing nullable-FK column to pattern-match** — the new
  `AppPageID` below is the first of its kind on this entity; style it like `AppPage.EntityType`/
  `MenuPosition` (nullable, conditionally-meaningful), not like a required FK.
- `RouteResolverService.resolve()` (`app-handlers-generic/src/services/RouteResolverService.ts`,
  lines 35-79): finds the section flagged `isPrimaryContentSection`, then picks **exactly one**
  widget from it — by `widgetID` exact match, else `formID` match, else `isDefault`, else first.
  `resolveFromUrl()` (81-143) does the identical single-widget selection. **This single-widget
  selection is precisely what changes to a multi-widget selection.**
- `AppPlayer.tsx`'s `getWidgetsForSection()` (lines 326-341): reads
  `engine.app.renderedWidgets: Map<sectionName, appWidgetID>` — one targeted widget ID per
  section, returns `[targeted]` or falls back to all untargeted widgets in the section. **This
  `Map<string, number>` shape is the other place needing a shape change** (to
  `Map<string, number[]>` or equivalent) for multi-widget-per-page rendering.
- `PagesPanel.tsx`: the "Target Widget" picker (lines 254-262) is a plain `<select>`, options
  built (116-121) from `appWidgets.filter(w => primarySectionNames.has(w.sectionName))` — **this
  is the exact complaint**: you can only pick from widgets already sitting in a primary-flagged
  section, so building a page means pre-populating that section first, then going to Pages to
  "claim" one of them.
- `AppNavMenu.tsx` `handleClick` (33-38): `engine.navigate({widgetID: page.appWidgetID,
  entityType: page.entityType})` — this stays correct for Physical/Virtual page navigation;
  what's new is menu items that AREN'T page navigation at all (Part 5).

## Part 0 — industry research: Wix/Webflow/Squarespace's page-first model, confirmed pattern

All three (and WordPress's block editor to a lesser extent) share the same information
architecture, which is the inverse of App Studio's current one:

- **The Pages list is the primary navigation of the editor itself**, not a config screen you
  visit after building content. Opening a site lands you on a page (Home, by default — a site is
  never page-less) with its own canvas.
- **The page IS the canvas.** Dragging a new element onto the canvas adds it to the page you're
  currently on. There is no separate "which page should this belong to" picker — placement and
  page-membership are the same action.
- **Site-wide chrome (header/footer/nav bar) is a separate, deliberately secondary editing
  surface** — "Edit Site Header," reached explicitly, not the default view, and not mixed into a
  page's own content list.
- **A menu item's target is explicitly polymorphic**, always at least: an internal page, an
  external URL (opens in the same or a new tab), and an anchor/scroll-to-section on the current
  page. Wix's own menu-item editor literally offers this as a type selector when you add an item.
  This directly matches Binoy's point 5 above — not a novel idea, it's the universal baseline.

Design conclusion: adopt this shape wholesale. The rest of this spec is the mechanical mapping
onto App Studio's existing entities/engine (`AppPage`, `AppWidget`, `RouteResolverService`,
`AppEngine.navigate()`) — no new routing system, per the same "don't rebuild what already works"
principle `app-studio-pages-navigation.md` established.

## Part 1 — data model: page-scoped widgets (resolved)

Add one nullable column to `AppWidget`:

```csharp
/// The Page this widget belongs to, when it lives in the primary content section. NULL means
/// "shared/default content" — shown when the app loads with no page targeted (mirrors the
/// pre-this-target behavior for every existing app/widget, so this is additive: every row today
/// implicitly becomes a shared/default widget, nothing breaks). A widget tagged with a PageID
/// only renders when that Page is the active navigation target.
///
/// Deliberately nullable, not a required FK: most widgets (header, nav, footer — anything NOT in
/// the primary content section) will never have this set at all, since only primary-section
/// widgets are ever page-scoped. FK -> AIExt_AppPages.AppPageID, ON DELETE SET NULL (deleting a
/// Page shouldn't cascade-delete its widgets — it un-scopes them back to shared/default, matching
/// how `AppPage.AppWidgetID`'s deprecation below un-scopes existing pages).
[ForeignKey(nameof(AppPage))]
public int? AppPageID { get; set; }
```

- **Only meaningful for widgets inside the section flagged `isPrimaryContentSection`.** A widget
  in any other section (header/nav/footer/sidebar) never has this set — those sections are
  app-wide chrome, not page content, per Part 0's "Layout Design vs Page Design" split.
- **`AppPage.AppWidgetID` is deprecated, not removed** (existing rows/consumers depend on it —
  removing it is a breaking schema change this target doesn't need to make). New pages created
  under this design leave it `0`/unset conceptually — a real non-nullable int column can't express
  "unset" cleanly, so: **migration decision** — either (a) widen it to nullable in the same
  migration (cleaner, matches the new semantics honestly), or (b) leave it required and
  auto-populate it with the Page's *first* tagged widget (in `DisplayOrder`) as a "primary/default
  widget within the set," kept in sync on every widget add/remove/reorder. **Recommend (a)** —
  honest nullability beats a synthetic auto-sync field nothing actually needs; confirm no other
  consumer (query, report, integration test) hard-depends on it being non-null before implementing
  (`AppPageServiceIntegrationTests.cs`/`AppPageRepositoryIntegrationTests.cs` almost certainly do
  and need updating).
- **Data migration for the 2 existing live pages** (created live tonight during the prior target's
  E2E pass, then cleaned up — confirm none remain in the real dev DB before writing this
  migration; if none remain, this step is a no-op documented for completeness, not skipped
  silently): for any `AppPage` row with a non-null `AppWidgetID`, set that widget's new
  `AppPageID` to the page's own ID. Backward-compatible: an app with zero migrated data still
  renders exactly as before (Part 2's resolution falls back to "no page targeted → shared/default
  widgets" when nothing is tagged).

## Part 2 — engine: primary section as an "outlet" resolving to a SET (resolved)

`RouteResolverService.resolve()`'s current single-widget selection becomes:

1. If `params.widgetID` (or `params.pageID` — see Part 5) targets an `AppPage`: return **every**
   `AppWidget` where `sectionName === primarySection.name && appPageID === page.appPageID`,
   ordered by `displayOrder`. Empty set is valid (a freshly-created, still-empty page) — the
   primary section renders empty, not an error, not a fallback to "all widgets."
2. If nothing is targeted (app just loaded, no page navigated to yet): return every `AppWidget` in
   the primary section where `appPageID IS NULL` (shared/default content) — this is the exact
   current no-page-targeted behavior, preserved byte-for-byte for backward compat.
3. `Virtual` pages: unchanged mechanism, still parameterize via `entityType`/`dataID` on top of
   whichever widget set step 1 resolves — a Virtual page's widget set is fixed, its *content*
   varies by the record ID, exactly as `app-studio-pages-navigation.md` Part 4 already specified.

`AppEngine`'s `renderedWidgets: Map<sectionName, appWidgetID>` (singular) becomes
`Map<sectionName, appWidgetID[]>` (or a small named type) — every read site (`AppPlayer.tsx`'s
`getWidgetsForSection`, `applyRouteTarget`) updates to the array shape. This is the one genuinely
invasive engine change in this target; everything else is additive. **No second routing system** —
`AppEngine.navigate()`'s public signature and `BrowserHistoryService`'s URL-push behavior are
unchanged; only what `RouteResolverService` hands back changes shape.

## Part 3 — Designer UX: page-first primary canvas (the actual "anti-usability" fix)

This is the structural change, not just a picker fix:

- **New default landing when an app is opened**: a Pages list (not the Section/Layout canvas).
  Every app has a Home page by default (create one automatically on app creation if none exists —
  matches "a site is never page-less"; also closes the pre-existing `AppPage.IsDefault` gap where
  nothing guaranteed one existed).
- **Selecting a page opens a canvas scoped to that page**: shows the full rendered app (chrome +
  content, so it's a real preview, not a fragment) but the primary content section is the only
  part that's "yours to edit" in this mode — adding a widget here **auto-tags it with the current
  page's `AppPageID`**, no picker. This is the "drag-to-build" ergonomic from the original
  discussion, riding on Part 1's simple nullable-column model rather than an implicit/hidden
  mechanism.
- **"Layout Design" becomes a separate, explicitly-reached secondary mode** — today's
  `CanvasPanel.tsx` Section list (Add Section, region assignment, non-primary-section widget
  management: header/nav/footer) moves here, scoped to only the non-primary sections (or shown
  read-only for the primary section, since that's page-controlled now). Reached via a clearly
  labeled toggle/tab, not the app's default view.
- **`PagesPanel.tsx`'s create-page flow drops the Target Widget `<select>` entirely.** Creating a
  page just asks Title/Slug/PageType/PageCategory/`ShowInMenu`+label+position — you get an empty
  page, then go build it (Part 3's page-scoped canvas). This directly fixes the reported
  complaint: no more "add all widgets to the content pane first, then pick one."
- **`AppConfigScreen.tsx`'s Widgets tab** (built earlier tonight) should show each widget's owning
  Page (or "Shared/Default" for `AppPageID IS NULL`) as a column/group — the one place that still
  benefits from a flat, page-agnostic list view (bulk auditing/troubleshooting), now with the new
  scoping visible instead of hidden.

## Part 4 — reconciliation with tonight's other builds (read before touching any of these)

- `AppConfigScreen.tsx` (General/Sections/Widgets tabs, full-screen, replaces the old Properties
  sidebar) **stays** — it's a secondary/admin surface (matches Wix's own separate "Site Settings"
  concept), not the primary canvas. Only its Widgets tab gains the Page-grouping above.
- `CanvasPanel.tsx` **splits**: its Section/Layout-editing half becomes "Layout Design" (Part 3);
  its widget-add/edit/remove machinery (`SectionCard`/`WidgetChip`/`AddWidgetModal`/
  `EditWidgetModal`) is reused as-is inside the new page-scoped canvas, just auto-tagging
  `AppPageID` on add instead of leaving it unset.
- `PagesPanel.tsx` **shrinks** (loses the widget picker) but keeps everything else (menu
  placement fields, reorder, delete, set-default).
- `SectionEditor.tsx`'s `isPrimaryContentSection` checkbox **stays exactly as-is** — still the one
  flag that marks which section is the page-scoped "outlet," still enforced at-most-one per
  layout.
- The already-open `AppEngine.navigate()` basePath bug (TODO doc, "Open" section) is unrelated to
  this target and stays out of scope — don't conflate the two.

## Part 5 — menu items as a polymorphic target (resolved)

Binoy: "not all menu items load widgets. some menu items will open a url on a new window, some
will just focus on a bookmark etc... keep various types of targets when designing menu item."

A menu-visible `AppPage` today implicitly means "target = this page's widgets." Generalize: a menu
item's target gets an explicit type, matching Wix's own menu-item type selector (Part 0):

```csharp
/// MenuTargetType: "Page" | "ExternalUrl" | "Anchor" — CHECK-constrained, same convention as
/// PageType/PageCategory/NavPosition. Only meaningful when ShowInMenu = true. Default "Page"
/// preserves every existing menu-visible AppPage's current behavior exactly (additive).
[StringLength(50)]
public string? MenuTargetType { get; set; } = "Page";

/// Only meaningful when MenuTargetType = "ExternalUrl". The literal URL to open.
[StringLength(2000)]
public string? ExternalUrl { get; set; }

/// Only meaningful when MenuTargetType = "ExternalUrl". Mirrors the standard target="_blank"
/// vs same-tab choice every site builder exposes for external links.
public bool OpenInNewWindow { get; set; } = true;

/// Only meaningful when MenuTargetType = "Anchor". The DOM id/data-anchor value to scroll to on
/// the CURRENT page — does not navigate via AppEngine.navigate() at all, a pure client-side
/// scrollIntoView, same as every anchor-link implementation.
[StringLength(200)]
public string? AnchorTarget { get; set; }
```

- **`MenuTargetType = "Page"`** (default): unchanged — `AppNavMenu`'s `onClick` calls
  `engine.navigate({widgetID/pageID, entityType})` exactly as today.
- **`MenuTargetType = "ExternalUrl"`**: `onClick` is `window.open(externalUrl, openInNewWindow ?
  '_blank' : '_self', 'noopener')` — zero engine involvement, a pure browser navigation. Note the
  existing `noopener` convention this codebase already uses for `DesignerToolbar.tsx`'s own
  `window.open` call (Preview button) — match it exactly for the same tab-nabbing-prevention
  reason.
- **`MenuTargetType = "Anchor"`**: `onClick` is `document.querySelector('[data-anchor="' +
  anchorTarget + '"]')?.scrollIntoView({behavior: 'smooth'})` — needs a way to actually MARK a
  spot in the primary section's rendered output with a matching `data-anchor` attribute; simplest:
  reuse `AppSection.name` itself as the anchor target for a "jump to this section" menu item
  (`data-section` already exists on every rendered `AppSectionRenderer` output per
  `LivePreviewPanel.tsx`'s own event-delegation code — confirm this attribute survives into
  `app-player`'s non-studio-mode render too, not just the Designer's `studioMode` overlay, before
  relying on it) — avoids inventing a second anchor-id system when one already exists on sections.
- **`AppPage.AppWidgetID`/the whole Physical/Virtual page machinery stays untouched for
  `MenuTargetType = "Page"`** — this is purely additive alongside it, not a replacement.
- **`PagesPanel.tsx`'s menu-config fields gain a target-type selector** — when `ShowInMenu` is
  checked, ask Page (existing flow) / External URL (URL + new-window checkbox) / Anchor (section
  picker, reusing the same section-name dropdown `RegionPicker`/`SectionEditor` already use
  elsewhere) before showing `MenuLabel`/`MenuPosition`/`DisplayOrder`.

## Constraints

- No auto-commits/pushes without explicit ask (already covered by standing instruction; each
  commit/push in this session so far was explicitly requested).
- `ID` uppercase in every new identifier (`AppPageID` on `AppWidget`, `MenuTargetType`, etc.).
- Additive-only wherever stated above — an app with zero migrated `AppPageID` data must render
  identically to today (Part 1/Part 2's fallback-to-shared-default behavior is the mechanism that
  guarantees this; verify it live, don't just assert it).
- No premature abstraction — `MenuTargetType` is a 3-value string enum matching the established
  `WidgetType`/`PageType`/`NavPosition` convention, not a polymorphic entity hierarchy.
- `pnpm typecheck`/`pnpm build` clean for every touched frontend package; `dotnet build` clean for
  every touched backend project.
- Dated `DevelopmentHistoryLog.md` entries in every touched package/project.
- Live-verify: create a Page with zero widgets, open its page-scoped canvas, add a widget (confirm
  it's auto-tagged and shows up ONLY on that page, not on other pages or the shared/default view),
  create a second Page and confirm its own separate widget set, add one menu item of each of the
  3 `MenuTargetType`s and confirm each behaves correctly (page swap / new-tab URL / same-tab
  scroll), confirm an app with pre-existing (pre-this-target) data still renders unchanged.

## Phase breakdown

### Phase 0 — verify, don't re-derive
Re-confirm every file/line snapshot above against current code (this session moves fast); check
whether any test/live data still references the deprecated single-widget `AppPage.AppWidgetID`
path before nullable-ing it.

### Phase 1 — backend
1. `AppWidget.AppPageID` nullable FK + migration (`ON DELETE SET NULL`).
2. `AppPage.AppWidgetID` → nullable (or resolve the (a)/(b) decision in Part 1 explicitly before
   implementing — don't default to whichever is easier without deciding).
3. `AppPage.MenuTargetType`/`ExternalUrl`/`OpenInNewWindow`/`AnchorTarget` columns + CHECK
   constraint + migration.
4. Data migration: existing `AppPage.AppWidgetID` → tag that widget's new `AppPageID`.
5. Update `AppPageService`/`AppWidgetService` validation for the new nullable/optional fields
   (mirror the existing conditional-validation pattern already used for `EntityType`/Virtual).

### Phase 2 — engine
1. `RouteResolverService.resolve()`/`resolveFromUrl()`: single-widget → widget-set resolution
   (Part 2).
2. `AppEngine`'s `renderedWidgets` map shape change + every read/write site.
3. `AppPlayer.tsx`'s `getWidgetsForSection` + `WidgetSlot`/`AppSectionRenderer` rendering a set.
4. `AppNavMenu.tsx`: branch on `MenuTargetType` (Part 5) instead of always calling
   `engine.navigate()`.

### Phase 3 — Designer UX
1. New page-first landing (Pages list as the default view when an app opens; auto-create a Home
   page for apps that have none).
2. Page-scoped canvas: opening a page shows the full app preview, primary-section widget
   add/edit/remove auto-tags `AppPageID`.
3. "Layout Design" secondary mode: relocate `CanvasPanel.tsx`'s Section/non-primary-widget
   management here, explicitly reached, not default.
4. `PagesPanel.tsx`: drop the Target Widget picker; add the `MenuTargetType` selector + its 3
   conditional field groups.
5. `AppConfigScreen.tsx`'s Widgets tab: group by owning Page / Shared-Default.

### Phase 4 — verification
Full live E2E per the Constraints section's bullet list, in both `app-studio-designer`'s
live-preview and `app-player`'s standalone runtime (same dual-registration discipline every prior
target in this codebase has followed).
