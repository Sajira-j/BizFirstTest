# App Studio — App Creation Flow (Project unification + template wizard)

Source: `app-studio-designer-components-react\src\{modals\CreateAppModal.tsx,toolbar\
DesignerToolbar.tsx,panels\{AppTreePanel,ProjectTreePanel}.tsx}`, `app-studio-api-client-js\src\
{apps,app-templates,projects}.client.ts`. Built 2026-09-04 (Task 22/23) — current as of that date.

## There is exactly one real way to create an App now

**"Create Project" auto-creates its App** — a two-call composition (`ProjectsApiClient.create()`
then `AppsApiClient.create({projectID})`, with a recovery path if the second call fails), routed
through the SAME wizard described below (the Project's name pre-fills into it). The standalone
"Create App" bypass that used to exist in two places (`AppTreePanel.tsx`'s "+ New App" and
`DesignerToolbar.tsx`'s top bar) has been removed — do not generate a flow that calls
`AppsApiClient.create()` directly without a Project; that path is intentionally gone.

`Project_Projects.ProjectTypeID` drives what gets auto-created: type `App` (ID 20) → an App Studio
app (current default, everything below). Type `Workflow` (ID 21) also exists in the seed data as of
this build for a future Flow Studio-process auto-creation path — check current code before assuming
that branch is wired end-to-end.

## The wizard (`CreateAppModal.tsx`)

Two real steps:
1. **Empty vs. Template** — a large visual choice.
2. **One combined page** for everything else: (if Template was chosen) a template gallery — live
   name-only cards fetched from `AppTemplatesApiClient` (no images — `Template_DataTemplates` has
   no image column) — then Name, Description, Industry, Category. **Create App is enabled the
   moment Name is filled** — the rest is optional, so a fast minimal flow and a fuller
   business-context flow are both "in one stretch," neither forced.

Industry/Category are NOT stored as new columns — they create/attach a real `Taxonomy_Records` row
(`BusinessIndustryID`/`BusinessCategoryID` among other Taxonomy fields) via the existing generic
Taxonomy API, set as the new App's `TaxonomyRecordID`. Don't invent a different storage shape for
these fields.

## Create Template from an App

`POST {base}/apps/{id}/create-template` — walks the App's real Pages/Widgets/AppWidgets and
serializes into one schema-versioned JSON document (internal temp-ID cross-references between
pages/widgets), inserted into `Template_DataTemplates.ContentData` under the pre-existing
`DataTemplateTypeID 50` ("New App" — no new template type was needed).

## Create App from a Template — the reverse, real ID semantics

`POST {base}/apps/create-from-template` — deserializes the template JSON and creates a BRAND NEW
App with brand new Pages/Widgets/AppWidgets. **Every App-owned structural entity gets a fresh ID —
nothing is copied.** Internal cross-references between cloned entities (a cloned Widget's PageID
pointing at a cloned Page) are remapped to the new IDs in one DB transaction.

**Widget-config ID references are classified, not uniformly kept or cleared** — this is the part
most likely to matter if you're generating or reasoning about a template-based app:
- **Kept as-is** (shared/library references, not part of what's being cloned): Form Widget's
  `formId` — the Atlas Form definition itself isn't cloned, so a form-bound widget in the new app
  still points at the SAME original form.
- **Cleared with a visible placeholder** (tenant-private data that would otherwise leak the
  original tenant's live data into the new app): Workflow Template Widget's
  `executionTemplateID`, and every media widget's (single + gallery) asset URL/reference fields.
- **Unclassified widget types**: flagged as a warning surfaced back to the user, rather than
  guessed at silently.

If you're generating a NEW template-cloneable widget type, its ID-shaped config fields need this
same classification decided explicitly (`WidgetConfigTemplateSanitizer` in the backend
`BizFirst.Ai.AppStudio.Api.Base\Services\` is where this logic lives) — don't assume a new field is
safe to keep just because it "looks like" `formId`.

## Deleting Apps

Every App create/delete flow goes through the same Project-scoped path now — a delete-orphan
gap that used to leave un-deletable Apps was closed in the same build (`AppTreePanel` now has a
real delete action). `DELETE {base}/apps/{id}` is a soft delete (per this codebase's general DB
convention — `Deleted` flag, not a hard row removal).
