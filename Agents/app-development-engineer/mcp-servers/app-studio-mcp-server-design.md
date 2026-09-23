# App Studio MCP Server — Design (v2, re-grounded 2026-09-09)

**Status: design only, ready to build.** Supersedes the 2026-09-04 version — App Studio's real API
surface changed materially in the interim (new endpoints, expanded contracts, one previously-excluded
capability now shipped). Every tool below is checked against the *current* controllers/services, not
the 2026-09-04 snapshot. Binoy's plan: review this today, then build `BizFirst.Ai.Mcp.Tools.AppStudio`
the same day, mirroring the proven `BizFirst.Ai.Mcp.Tools.AtlasForms`/`.Workflow` pattern.

## What changed since v1 (read this first)

- **Route base moved**: everything is under `api/v1/app-studio/*` now, not bare `/apps/*`.
- **Style editing is no longer excluded — it's real.** `UpdateAppWidgetRequestDto` has a plain
  `StyleConfiguration` field (JSON, slot → `{css?, style?}`), plus `WidgetStyle`/`WidgetCss`. v1's
  "no style-editing endpoint exists" is no longer true; this is now just a tool parameter.
- **Section/layout editing has a real, safe path now**: `POST /apps/{id}/apply-starter-action` applies
  an `AppStudioStarterAction` template (creates real Widget+AppWidget rows, **appends** section(s) into
  `App.Configuration`, never overwrites, one transaction, optimistic concurrency on `LastModifiedOn`).
  Not free-form section editing (still doesn't exist), but a genuine, template-driven way to add page
  content via MCP today.
- **Publish is unchanged and still real** (`POST /apps/{id}/publish`, `AuthorizeTenantAdminAttribute`,
  snapshots an `AppVersion` with `IsPublished=true`) — known backend gap: doesn't unpublish prior
  versions. The v1 "should an AI publish?" question is resolved below, not left open.
- **New capability with no v1 equivalent at all**: app-code assignment (`GET /check-code`,
  `PATCH /{id}/app-code`), full `AppVersion` CRUD + `restore`, `AppStudioSettings` (tenant-level
  key/value config), a generic bare `create_app` (not just template-based), full-object `update`
  endpoints for App/Page/AppWidget/Widget (not partial-field), `DELETE` on Page/AppWidget/Widget
  (v1 only had `delete_app`).
- **Widget count**: RAG's own overview table has always listed **18** widget types — the "17" in v1
  of this doc was this document's own typo, not a code change. No new widget type was added.

## Real route surface (confirmed against current controllers)

| Controller | Route base | Service |
|---|---|---|
| `BaseAppStudioAppController` | `api/v1/app-studio/apps` | `IAppService` (AIExtension) + `IAppStudioAppService` |
| `BaseAppStudioAppPageController` | `api/v1/app-studio/apps/{appID}/pages` | `IAppPageService` |
| `BaseAppStudioAppWidgetController` | `api/v1/app-studio/apps/{appId}/widgets` | `IAppWidgetService` |
| `BaseAppStudioWidgetController` | `api/v1/app-studio/widgets` | `IWidgetService` |
| `BaseAppStudioAppVersionController` | `api/v1/app-studio/apps/{appId}/versions` | `IAppVersionService` |
| `BaseAppStudioSettingsController` | `api/v1/app-studio/app-settings` | `IAppStudioSettingsService` |
| `BaseAppStudioAppAuditLogController` | — | read-only audit trail; **out of scope**, not a builder capability |

`IAppService`/`IAppPageService`/`IAppWidgetService`/`IWidgetService`/`IAppVersionService` live in
`BizFirst.Ai.AIExtension.Domain`/`.Service` — the App entity genuinely belongs to AIExtension, reused
by App Studio. `IAppStudioAppService` (Publish/ApplyStarterAction/tenant-scoped listing) is App
Studio's own — see the layering decision below for why that matters.

## Curated tool set — V1 (build today)

Curated per the standing rule (task-shaped tools, not 1:1 with endpoints) — 15 tools, in line with
Workflow's 12 and Forms' 8. Destructive/higher-risk/newer capability is deliberately deferred to V2
(below), not because it's unbuildable today but because a first pass should prove the core
describe-an-app → get-an-app loop before adding surface area.

| Tool | Wraps | Params | Notes |
|---|---|---|---|
| `list_apps` | `POST /list` or `POST /by-project` | `projectID?` | One tool, branches internally on whether `projectID` is supplied — collapses two real endpoints into one task-shaped read, per the standing read-collapsing rule. |
| `get_app` | `POST /get-by-id` or `GET /by-code/{code}` | `appID?`, `appCode?` (exactly one required) | Same collapsing approach. |
| `create_app` | `POST /apps` | `name`, `description?`, `projectID` | Generic create against an existing project. |
| `create_project_with_app` | `ProjectsApiClient.create` + `POST /apps` | `name`, `description?`, `projectType`, `industry?`, `category?` | Composite — creates a **new** project, not just an app. Genuinely different from `create_app`, keep both. |
| `create_app_from_template` | `POST /create-from-template` | `templateId`, `name`, `description?`, `projectID?`, `taxonomyRecordID?` | Two new optional params vs. v1 — surface `warnings` in the response, don't swallow them. |
| `update_app` | `PUT /{id}` | `appID`, `name?`, `description?`, `appCode?` | Full-object endpoint in real code; tool exposes only the safe subset an agent should touch. Folds app-code assignment in rather than a separate tool — check-availability happens server-side, return a clear conflict error if taken. |
| `delete_app` | `DELETE /{id}` | `appID` | Soft delete, unchanged from v1. |
| `list_pages` | `GET /pages` | `appID` | |
| `create_page` | `POST /pages` | `appID`, `name`, `title`, `slug`, `parentPageID?` | Real endpoint takes a full `AppPage` object; tool exposes the fields an agent actually sets. |
| `update_page` | `PUT /pages/{id}` | `appID`, `pageID`, plus the handful of fields an agent plausibly edits (`name?`, `title?`, `slug?`, `showInMenu?`, `menuLabel?`, `displayOrder?`, `isDefault?`) | Real endpoint has 16 optional fields; folding reorder (`displayOrder`) and set-default (`isDefault`) into this one update rather than 3 separate tools — they're just field writes on the same entity. |
| `delete_page` | `DELETE /pages/{id}` | `appID`, `pageID` | |
| `list_widgets_in_app` | `GET /widgets` | `appID` | Placements, not definitions. |
| `create_widget` | `POST /widgets` (definition) + `POST /apps/{appId}/widgets` (placement) | `appID`, `widgetType` (one of 18 real values), `name`, `configuration`, `sectionName`, `appPageID?` | Two-call composite, unchanged shape from v1. |
| `update_widget_placement` | `PUT /apps/{appId}/widgets/{id}` | `appID`, `widgetPlacementID`, `styleConfiguration?`, `widgetStyle?`, `widgetCss?`, `displayOrder?`, `showInNav?`, `navPosition?` | **New vs. v1**: this is where style editing now lands — no longer excluded. Named `_placement` (not `_config`) to distinguish from the widget definition below. |
| `list_widget_types` | Static `WIDGET_TYPE_REGISTRY` | — | 18 rows, not 17 (v1 typo fixed). |

## Deferred to V2 (real capability, not blocking today)

- `delete_widget_placement` / `delete_widget_definition` — the latter especially: a `Widget` definition
  can in principle be referenced by more than one placement, so deleting it needs a reference check
  first; not a same-day scope.
- `update_widget_definition` (`PUT /widgets/{id}`, full `Widget` object incl. type-specific config) —
  real and useful, but `update_widget_placement` covers the common "restyle this instance" case; the
  shared-definition edit case can follow once the placement path is proven.
- `publish_app` — real, resolved to **`HumanOnly`** (same tier as `create_credential`/`create_agent`/
  `create_mcp_server`), consistent with treating "make this live" as a human-gated action, not an
  agent-writable one. Build it, but don't let it block today's core-loop work.
- `apply_starter_action` — genuinely useful (the safe section/content-append path), but adds
  section-tree/transaction complexity; sequence after the flat App/Page/Widget loop is certified.
- `create_template_from_app` — real, low-risk, just not core-loop-critical for a first pass.
- App versioning (`list`/create snapshot/`restore`) and `AppStudioSettings` — real, separate concerns
  from "build an app"; natural second module pass, not part of the builder core loop.

## Per-widget-type strategy — one generic tool, never one tool per widget type

18 real widget types, each with a different `configuration` shape, is the same shape of problem
Workflow Nodes RAG (107 node types) and Atlas Forms RAG (per-control-type schemas) already solved —
both times with one generic tool plus a two-tier knowledge base, never a tool per variant. Same answer
here, for the same reasons (tool-catalog explosion measurably hurts LLM tool-selection accuracy; one
code path is what actually gets secured/audited). **`app-studio-rag\widgets\{type}.md` already has
full first-pass Tier 1 coverage for all 18 types** — ahead of where Workflow Nodes RAG is (18/107) —
so the knowledge side of this is already done; it just isn't wired into the MCP tool's own guidance
yet. Concretely:

- `create_widget`/`update_widget_placement` stay the only two tools, for every widget type.
- The calling agent is expected to fetch `widgets\{type}.md` for a type it hasn't already seen before
  constructing `configuration` — this is a documentation/procedure note to embed in the tool module's
  own description, not a new capability to build.
- **Exception, ergonomics only**: inline one compact `content`-widget example directly in
  `create_widget`'s `[Description]`, since it's the simplest config (no cross-reference) and will be
  the single most frequent call by volume — skips a RAG fetch for the common case without creating a
  second tool.
- **Cross-domain reference widgets need an extra validation step, not an extra tool.** `form`
  (`formID`), `workflow-template` (`executionTemplateID`), `workflow-template-category`
  (`executionTemplateCategoryID`), and `chat-panel` (`processID`) each reference an entity that must
  already exist in another domain (Atlas Forms / Flow Studio). The agent creates that entity first via
  the Forms/Workflow MCP tools, then passes its ID in — App Studio's tools never call into another
  domain's service directly. Server-side, `create_widget`'s validation for these four types must
  confirm the referenced ID both **exists** and **belongs to the caller's own tenant** before accepting
  the widget, not just that the config shape is well-formed — an unchecked foreign ID here is a
  cross-tenant data-integrity risk, the same class of gap the "TenantID never client-supplied" rule
  exists to prevent, one hop further out.

## Resolved open questions from v1

1. **Config validation strictness** — validate `configuration` server-side against each widget type's
   real schema before storing (same lesson the Workflow RAG doc already learned:
   `ConfigurationSchema`-shaped columns can't be trusted blind, and an LLM needs real error feedback,
   not a silent bad write). For the four reference-holding types, this validation also includes the
   existence + tenant-ownership check above, not just shape-checking.
2. **Publish-and-verify** — build it, gate it `HumanOnly`. Resolved, not deferred as a policy question.
3. **One server vs. split** — one module, `BizFirst.Ai.Mcp.Tools.AppStudio`. Matches the existing
   one-module-per-domain convention exactly.

## New open decision — service layering (needs Binoy's call before coding starts)

The proven pattern (`Mcp.Tools.AtlasForms`/`.Workflow`) is: the tools project references only lean
**Domain**/**Extended.Domain**/**Extended.Services** projects (POCOs + interfaces, no ASP.NET Core
framework reference) plus `Go.Essentials.Domain` — never a web/API project. App Studio doesn't cleanly
fit that today:

- `IAppService`/`IAppPageService`/`IAppWidgetService`/`IWidgetService`/`IAppVersionService` **do** live
  in a clean `BizFirst.Ai.AIExtension.Domain`/`.Service` split — these are fine to reference directly,
  same as Atlas Forms references `IFormService`.
- `IAppStudioAppService` (Publish, ApplyStarterAction, tenant-scoped app listing) — the App-Studio-
  specific capability, the closest analog to Atlas Forms' `IFormsExtendedService` — **lives inside
  `BizFirst.Ai.AppStudio.Api.Base` itself**, a project with
  `<FrameworkReference Include="Microsoft.AspNetCore.App"/>` and real MVC controllers in it.

**Option A — recommended.** Extract `IAppStudioAppService` (and its implementation) out of
`AppStudio.Api.Base` into a new slim `BizFirst.Ai.AppStudio.Extended.Domain`/`.Extended.Services`
project pair first (mirrors the existing `*.Extended` convention this codebase already uses
elsewhere, e.g. Atlas Forms' own `.Extended.{Domain,Services}`), then build `Mcp.Tools.AppStudio`
against that + `IAppService`/etc. directly. Slightly more upfront work today, but keeps the module
consistent with every other domain's precedent and doesn't couple an MCP tools library to a full web
framework reference.

**Option B — faster, not recommended as the default.** Reference `BizFirst.Ai.AppStudio.Api.Base`
directly from `Mcp.Tools.AppStudio` today. Works, ships faster, but pulls the full ASP.NET Core App
framework and every controller into a project that's supposed to be a thin, framework-free tool
wrapper — breaks the pattern every other module follows, and the DI/controller registration overlap
this creates hasn't been evaluated.

Recommend Option A unless today's timeline can't absorb the extraction — flag which one to build
against when you review this.

## Auth / scoping — needs a new scope added, code change required

Confirmed (separate investigation, same session): valid MCP scope strings live in a hardcoded
in-memory catalog, `BizFirstFi.Go.IAM.Service\Services\StaticApiKeyScopeCatalog.cs`, currently listing
`mcp:atlas-forms:{read,write}`, `mcp:credentials:{read,write}`, `mcp:workflow:{read,write}` — **no
`mcp:app-studio:*` entries exist**. Adding this module requires appending
`mcp:app-studio:read`/`mcp:app-studio:write` to that catalog (a one-line-per-scope C# change, no DB
migration — there is no DB-backed scope table today) before any API key can be granted access to it.
`TenantID` is never an LLM-supplied parameter, same rule as every other module — resolved server-side
from the caller's auth context.

## Real Mcp.Tools project pattern to mirror exactly

Confirmed from `BizFirst.Ai.Mcp.Tools.AtlasForms`:

- Project references: `ModelContextProtocol` NuGet only, plus `ProjectReference`s to the domain's
  Domain/Extended.Domain/Extended.Services projects and `Go.Essentials.Domain`. No ASP.NET Core
  framework reference, no MCP host of its own.
- One `static class` per tool, `[McpServerToolType]`.
- One `static async Task<string>` method per tool, `[McpServerTool(Name = "...")]` + `[Description]`
  on the method; every parameter individually `[Description]`-annotated (this is what the SDK uses to
  auto-generate the schema — never hand-write a schema).
- The backing service interface and a trailing `CancellationToken` are method-injected (ASP.NET Core
  DI-style parameter injection), the method body calls the service directly, and the return value goes
  through the shared `McpToolResponse.FromResponse(...)` helper — reuse this helper, don't reinvent
  response shaping per module.

## Sequencing

1. Resolve the service-layering decision (Option A recommended).
2. Build `Mcp.Tools.AppStudio` with the 15 V1 tools above.
3. Add the two new scopes to `StaticApiKeyScopeCatalog.cs`.
4. Register the module in `PlatformWebServerExtensionsByBuilder.cs`'s existing `AddMcpServer()` call
   (one more `.WithToolsFromAssembly(...)`, no new host).
5. Certify the same way Atlas Forms was: live `tools/list`, then a real create → read → update → delete
   round trip against live data, confirmed both in the tool response and directly in the database.
