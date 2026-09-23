# App Studio — Core Data Model & API

Source: `app-handlers-core\src\types\{WidgetRecord,AppPageRecord,AppSection}.ts`,
`app-studio-api-client-js\src\{apps,pages,app-widgets,widgets}.client.ts`.

## Hierarchy

```
App (AppRecord)
 └─ AppPage (AppPageRecord)      — a page; parentPageID for nesting
     └─ AppSection                — a layout region on a page (header/nav/footer/primary content)
         └─ AppWidget (AppWidgetRecord) — one PLACEMENT of a Widget in a section on a page
             └─ Widget (WidgetRecord)    — the shared, reusable widget DEFINITION + its config
```

**The Widget/AppWidget split is the one thing to never get wrong**: `Widget` (`WidgetRecord`) is
the shared definition — `widgetID`, `widgetType`, `configuration`, `name`. `AppWidget`
(`AppWidgetRecord`) is one PLACEMENT of that Widget into a specific App/Page/Section —
`appWidgetID`, `appID`, `widgetID` (FK to the Widget), `sectionName`, `appPageID`, `displayOrder`,
plus placement-only fields (`isDefault`, `showInNav`, `navPosition`, `routable`). The SAME Widget
definition can be placed more than once (multiple AppWidget rows referencing one widgetID) — editing
the shared Widget's config affects every placement; placement-only fields (`showInNav`, etc.) are
independent per placement. `AppWidgetRecord.configuration` can also hold a placement-level override
that layers on top of the Widget's own `configuration` — check the specific widget's config doc for
whether it supports instance-level overrides (Form Widget's `formOverrides` is the clearest example).

## CRITICAL GAP (confirmed live, 2026-09-10): `create_widget`'s `sectionName` does nothing unless that section already exists

`AppSection` is a real, separate, explicitly-created object — it is **not** conjured into existence
just because an `AppWidget` row references its name. Confirmed by live reproduction: calling
`create_widget` with `sectionName: "main"` (and `"hero"`, `"header"`) against a brand-new app
succeeded and wrote real `AppWidget` rows, but the App Studio Designer showed **"No layout defined
yet" / "No sections yet"** for every page — the widgets were silently orphaned. Opening the
Designer's "Site Structure" panel and clicking "+ Add Section" confirmed sections are a distinct,
separately-created concept (each gets an auto-generated name like `app-{appID}-section-{timestamp}`
until manually renamed); renaming a newly-created section to a name that matches existing orphaned
`AppWidget.sectionName` values immediately attached them (all widgets sharing that section name
across every page attach to the one section — per-page scoping still happens via each widget's own
`appPageID`, not via a separate section-per-page).

**As of this writing, none of the 15 App Studio MCP tools can create an `AppSection`.** This means
an app built purely through `create_widget` calls, with no supplementary UI step, has widgets that
exist in the database but never render anywhere — a first-run agent will not discover this from any
error message, since every `create_widget` call reports success. This is the single highest-priority
gap in the App Studio MCP module: until a `create_section`/`add_app_section` tool (or an equivalent
"ensure this section exists" auto-create behavior inside `create_widget`) ships, App Studio MCP
cannot build a genuinely working app end-to-end without a human bridging this one step in the
Designer UI.

Separately, also confirmed live: even with sections correctly wired, the real end-user Player
(`app-player`, not the Designer canvas) only rendered the shared `header` section's widget (page
navigation) — the page-scoped `hero`/`main` section content did not appear on the Player's rendered
page in this session's test, despite rendering fine in the Designer's per-page canvas view.

**UPDATE (2026-09-10, follow-up session) — `create_section` tool now ships (below), and
`isPrimaryContentSection` alone is confirmed NOT sufficient to fix this.** Real repro: created a new
section (`content-primary`) with `isPrimaryContentSection: true` via the new tool, placed a real
`content` widget in it with `appPageID` set to the Home page, confirmed via direct SQL that
`App.Configuration` stored exactly the expected shape
(`{"name":"content-primary","isPrimaryContentSection":true}` alongside the pre-existing sections) —
then loaded the real Player, both via a bare `/{appID}` URL and via an actual in-app nav click
(`/​{appID}/page/home`, confirmed via the URL bar that this really does trigger
`RouteResolverService.resolve()`'s page-anchored branch, not just the bare-root default-widget
branch). **The widget still did not render — page body content remains completely blank on the real
Player, nav bar aside, in both cases.** `RouteResolverService.findContentPaneSection()`/`resolve()`
(`app-handlers-generic\src\services\RouteResolverService.ts`) does read `isPrimaryContentSection`
and a real passing test (`AppPlayer.pageScopedWidgets.test.tsx`) proves the mechanism works under
test conditions — so the gap is somewhere between "real live data reaches the Player" and "the
resolver runs against it": prime suspects, not yet checked, are (a) the Player/AppEngine caching
`appWidgets`/`configuration` from an earlier load and never refetching after the section/widget were
added via MCP mid-session (no observed network refetch on nav-click), or (b) some other loader-level
filter silently excluding widgets created after initial app load. **This is now the single most
important remaining App Studio gap** — until resolved, an app built and modified entirely via MCP
tools cannot be confirmed to render its actual content for a real end user, only its shared nav.
Needs a dedicated follow-up with browser devtools network/console inspection (not yet done in this
pass) rather than more trial-and-error section/widget creation.

**RESOLVED (2026-09-10, same night, third follow-up pass) — it was never a resolver/caching bug.**
Both prime suspects above were wrong. Confirmed via a genuine hard reload (F5, screenshotted before
and after) that did NOT fix it — ruling out client-side staleness — then via direct DOM inspection
(`document.querySelector('[data-widget="184"]')`) that the widget WAS mounting in the resolved
section (a real `<style>` tag + wrapper, matching `ContentWidgetRenderer`'s own structure), just
rendering zero visible text. That pointed at the widget's own config, not the layout pipeline.
**Real root cause**: `widget-handlers-content-widget`'s actual `ContentWidgetConfig` requires
`{ content: string, format: 'html'|'markdown'|'text' }` — two separate fields — but
`create_widget`'s own server-side `[Description]` (`CreateWidgetTool.cs`) showed a WRONG example, a
flattened `{"markdown": "..."}` shape, and `WidgetTypeCatalog.Validate()`'s "content" case checked
for exactly that wrong shape too, so it was silently accepted and stored, then rendered as nothing.
The RAG correction made earlier the same night (see `widgets\content.md`'s own "History of this
doc's own bug" section) had itself gone the wrong direction — it "confirmed" the wrong shape against
the buggy validator instead of the real renderer. All three now fixed at the source:
`CreateWidgetTool.cs`'s example, `WidgetTypeCatalog.cs`'s content-type validation, and
`content.md` itself. Live-verified: a widget created with the correct `{content, format}` shape
rendered real visible text in the Player on the very next reload, and a follow-up
`update_widget_placement` call (padding/max-width via `styleConfiguration.widgetContainer.style`)
proved the full design-review-and-fix loop works end to end via MCP alone — see
`test-results\app-studio-mcp-coverage\app-studio-mcp-coverage.html` for the before/after screenshots
and every real request/response. **The lesson for future gap-hunting in this codebase**: when a
tool's validator and its actual renderer disagree about a config shape, trust the renderer — it's
what an end user actually sees, and a validator can be exactly as wrong as a doc.

**FOLLOW-UP (2026-09-11, independent session)** — the `{content, format}` source-level fix above did
stick (independently re-verified via a fresh `tools/list` call: `create_widget`'s example is
correct). But the fix was never paired with a data-migration/audit pass, so **every widget created
BEFORE that fix landed kept the old broken `{"markdown": "..."}` shape indefinitely** — found 10 such
widgets still live across 2 different apps (including a real client-facing demo app, rendering
completely empty on every page) a full session later. **Lesson: fixing a tool's validator/generator
does not retroactively fix already-created data — after any such source-level config-shape fix, run
a one-time audit query across the ENTIRE existing dataset for the old wrong shape** (e.g.
`WHERE JSON_VALUE(Configuration, '$.markdown') IS NOT NULL` for this specific case) and correct it,
don't assume "the bug is fixed" means "existing data is fine."

**Separately, a genuinely different bug from the config-shape one** (same symptom — blank Player
page — different cause): real page content can be built entirely correctly (right `{content,
format}` shape, right `appPageID`) and still be invisible if it's placed in a section that ISN'T the
one `layout.appSections` entry flagged `isPrimaryContentSection: true` for that specific app. Section
names used for this role have varied across apps built at different times (`main`, `main-content`,
`content-primary` all seen in the wild) — **always read `AIExt_Apps.Configuration`'s
`layout.appSections` first to find which section name is actually flagged primary for THIS app**,
never assume by name. `AppPlayer.tsx`'s `getWidgetsForSection()` shows only `appPageID == null`
widgets for every section that ISN'T the flagged primary one, unconditionally — a widget with a real
`appPageID` sitting in the wrong section is invisible with no error on either the write or read side.

**Also (2026-09-11)**: a bare app-root Preview URL (`/{appID}?tenantID=...`, no `/page/{slug}`
segment — exactly what clicking "Preview" from the Designer produces) resolves differently from an
explicit `/page/{slug}` deep link. `RouteResolverService.resolve({})` (no widgetID/formID/pageID)
used to pick `paneWidgets[0]` — an arbitrary widget by raw `displayOrder` across every page, with no
concept of the app's own configured home page — instead of consulting `AppPage.isDefault`. If that
arbitrary first widget happened to be an orphaned `appPageID: null` leftover, the bare Preview link
showed nothing while an explicit `/page/home` link on the IDENTICAL app worked fine — always test
both URL forms, they are genuinely different code paths that can diverge. Fixed by preferring
`AppPage.isDefault` when no anchor is given at all (`RouteResolverService.ts`'s `resolve()`).

**Designer canvas gotcha**: dropping a widget onto the canvas's primary section did not stamp the
currently-open page's ID onto the new placement (landed `appPageID: null`), so newly-added page
content appeared in the Section Details widget list (proving the placement was created) but never
rendered on canvas, even after a hard reload — fixed in `CanvasPanel.tsx`'s drop handler. Native
HTML5 canvas drag-and-drop also cannot be driven by browser automation (confirmed: neither a
simulated mouse drag nor a full synthetic `DragEvent` sequence had any effect, while a real human
mouse drag in the same session worked) — for automated testing, use the click-based "+ Add Widget"
modal instead (now available even on primary/page-controlled sections after this session's fix; it
already correctly threads `appPageID` through end-to-end).

## `AppWidgetRecord` fields worth knowing

| Field | Meaning |
|---|---|
| `appPageID` | `null` = shared/default content shown when no page is targeted (every pre-existing AppWidget row is implicitly this). Only meaningful inside the section flagged `isPrimaryContentSection` — a header/nav/footer widget never sets this. |
| `isDefault` | true = this is the default widget for its section; `widgetID=0` in the URL resolves here. |
| `showInNav` / `navPosition` | Whether/where a nav link for this widget appears (`'top'` \| `'side'` \| `null`). |
| `routable` | Whether activating this widget updates the browser URL. |
| `styleConfiguration` | Level-3 style slots, independent per placement — two placements of the same Widget can look different. |

## `AppPageRecord` — key field

`parentPageID` (nullable, self-referencing FK) — `null`/`undefined` = top-level page. Drives the
`page-navigation` widget's expandable tree. The legacy fixed nav bars ignore this and show a flat
list; a page-navigation widget on the app replaces them (`AppPlayer.tsx` stops rendering the legacy
bars once one is placed).

## Real CRUD API (base path pattern: `{apiBaseUrl}/apps/...`, App Studio's own module)

| Action | Endpoint |
|---|---|
| List apps | `POST {base}/apps/list` |
| Get app | `POST {base}/apps/get-by-id` |
| List apps by project | `POST {base}/apps/by-project` |
| Create app | `POST {base}/apps` |
| Update app | `PUT {base}/apps/{id}` |
| Delete app | `DELETE {base}/apps/{id}` (soft delete — see `app-creation-flow.md` on orphan cleanup) |
| Publish app | `POST {base}/apps/{id}/publish` |
| Create template from app | `POST {base}/apps/{id}/create-template` — see `app-creation-flow.md` |
| Create app from template | `POST {base}/apps/create-from-template` — see `app-creation-flow.md` |
| List pages | `GET {base}/apps/{appId}/pages` |
| Create page | `POST {base}/apps/{appId}/pages` |
| Update page | `PUT {base}/apps/{appId}/pages/{id}` |
| Delete page | `DELETE {base}/apps/{appId}/pages/{id}` |
| Reorder pages | `POST {base}/apps/{appId}/pages/reorder` |
| Set default page | `POST {base}/apps/{appId}/pages/{id}/set-default` |
| List placements (AppWidgets) for an app | `GET {base}/apps/{appId}/widgets` |
| Create placement | `POST {base}/apps/{appId}/widgets` |
| Update placement | `PUT {base}/apps/{appId}/widgets/{id}` |
| Delete placement | `DELETE {base}/apps/{appId}/widgets/{id}` |
| Set default placement | `POST {base}/apps/{appId}/widgets/{id}/set-default` |
| List/get/create/update/delete Widget definitions | `{base}/widgets`, `{base}/widgets/{id}` (standard REST verbs) |

`Project` is a SEPARATE backend module (`/api/v1/project/projects/*`, `BizFirst.Ai.Project.Api.Base`
— not App Studio's own `/apps/*`). See `app-creation-flow.md` for how Project and App now compose.

## Gotchas

- `configuration` is stored as arbitrary JSON on both `Widget` and `AppWidget` — every widget
  config interface declares an index signature (`[key: string]: unknown`) specifically so
  `resolveConfig<T>()` can read it generically. Don't assume a config object is exhaustively typed
  by its interface at runtime — extra keys are tolerated.
- Widget creation UI is driven entirely by `WIDGET_TYPE_REGISTRY`
  (`app-handlers-core\src\types\WidgetTypeRegistry.ts`) — this is the single source of truth for
  what widget types the creation picker offers. Adding a 6th... 18th widget type needs an entry
  there PLUS a real handler package — the registry alone doesn't make a type actually render.
- Some widget types are "safe defaults" (drag-to-instant-place with sensible zero-config behavior);
  most single-media and binding-required types (`form`, `image`, `video`, `audio`, `pdf`,
  `chat-panel`, `workflow-template`, `workflow-template-category`) are NOT — they always go through
  a creation modal because they have a hard-required field with no sensible default. Check each
  widget's own Tier 1 doc for whether it's safe-default or modal-required.
