# Target: App Studio — workflow-template / workflow-template-category / chat-panel widget types

## Problem statement

App Studio (`BizFirstAiStudio\src\app-studio`) has two widget types today (`form`, `content`),
each a self-contained `widget-handlers-<type>-widget` package registered into a `WidgetRegistry`
consumed by both the studio's own live-preview canvas and the standalone `app-player` runtime.
Binoy wants three more, following that exact pattern:

1. **`workflow-template`** — config takes one `ExecutionTemplateID`. Renders as a single card
   (name/description/icon or image, an Execute or Chat-Now action depending on the template's
   type) — reusing the **already-built** `@bizfirst/ai-agent-catalog-ui` package's `TemplateCard`,
   not rebuilding one.
2. **`workflow-template-category`** — config takes one `ExecutionTemplateCategoryID`. Renders the
   list of templates in that category as a grid of the *same* cards `workflow-template` uses — no
   duplicated card component.
3. **`chat-panel`** — embeds a chat window (ChatDesk's real feature set) into an App Studio app.
   Reuses ChatDesk's real `ChatWindow`/`ChatWindowContainer` components — which today live
   app-local inside `chatbots\apps\chatdesk\src\components\` and are **not yet a package**; per
   Binoy's "put everything in reusable packages" instruction and this component pair's own
   existing doc comment ("a later extraction into a standalone reusable package — the directive's
   explicit long-term goal"), extracting them is real, necessary, in-scope work for this target,
   not optional polish.

Plus a fourth, cross-cutting piece: **a widget-type alias/display-name system**. `AddWidgetModal.tsx`
today hardcodes a `'new' | 'existing'` tab and, within "New", a hardcoded `'form' | 'content'` radio
pair with literal labels `"Form Widget"` / `"Content Widget"`. It's about to grow from 2 to 5 types.
Binoy's concrete example: the internal widget type `workflow-template` (backed by the
`ExecutionTemplate` entity) should show to the end user as **"Workflow Agent"**, not the raw type
name. Design a small per-widget-type alias registry and make the picker data-driven over it,
instead of hand-adding a third/fourth/fifth hardcoded radio row.

## Prior analysis (read this first — don't re-derive from scratch)

**This is mostly NOT green-field.** A prior, Binoy-approved design pass already exists and a real
package was already built from it — confirm its current state (see "real gaps" below) rather than
assuming it's finished or that it's stale:

- `Documentation\Employees\agentic-coding\ai-agent-browser-spec\` (`overview.md`,
  `architecture.md`, `api-contracts.md`) — the original design spec for a browsable AI-agent/
  workflow-template catalog. Read this first; it's the real precedent for "card for one template,
  category list for many, click-to-execute or click-to-chat."
- `BizFirstAiStudio\src\bizfirst-common\ai-agent-catalog-ui\` — built from that spec
  (`DevelopmentHistoryLog.md` dated 2026-08-19, "not redesigned here", "nothing committed"). This
  package **already implements almost everything** `workflow-template`/`workflow-template-category`
  need:
  - `src/components/TemplateCard/TemplateCard.tsx` — the exact card: icon/image, name/description,
    an **Execute** button (calls `client.executeTemplate(executionTemplateID)`) for a normal
    template, or a **Chat Now** button (resolves `{appID, processID}` via
    `client.resolveChatTarget(executionScope, scopeReferenceID)` and navigates to
    `{chatDeskBaseUrl}/appid/{appID}/processid/{processID}?executionTemplateID={id}`) for a
    conversational one. Chat-type detection: `isChatExecutionTemplateType()` in `types.ts`.
  - `src/components/TemplateCatalog/TemplateCatalog.tsx` — composes `CategoryList` + a
    `TemplateCard` grid; this is close to `workflow-template-category`'s shape but is the
    *browse-all-categories* UI (a sidebar of every category). `workflow-template-category`'s own
    handler should call the lower-level hook directly (next bullet) rather than mounting the whole
    multi-category browser for one fixed `ExecutionTemplateCategoryID`.
  - `src/hooks/useExecutionTemplatesByCategory.ts` — `(executionTemplateCategoryID) => {templates,
    loading, error, refresh}`. Exactly `workflow-template-category`'s data need.
  - `src/api/aiAgentCatalogApiClient.ts` (`AiAgentCatalogApiClient`) — methods: `getEnabledCategories`,
    `getEnabledTemplateTypes`, `getTemplatesByCategory`, `executeTemplate`, `resolveChatTarget`.
    **No `getTemplateById` method exists yet** — see "Real gap #1" below; `workflow-template`
    (single-ID) needs one.
  - `src/context/AiAgentCatalogApiContext.tsx` (`AiAgentCatalogApiProvider`/
    `useAiAgentCatalogApiClient`) — host-configurable `baseUrl`/`getAuthToken`/`tenantId`, same
    shape as `conversations-react`'s provider. A widget handler's `Component` needs to be able to
    reach a configured client — either the App Studio app already wraps its tree in this provider
    (check `app-player`'s `App.tsx` / app-studio-designer's `AppShell.tsx`, neither does today) or
    the new widget package must construct/provide its own instance using
    `WidgetRenderContext`'s `tenantID` + whatever auth-token source `app-player`/studio already use
    (`getToken`/`useAuthStore`, see files in scope below).
  - Real backend endpoints already wired and auth-protected (`BaseExecutionTemplateController.cs`,
    `[AuthorizeRegularUserAttribute]`), confirmed by the package's own dev log, not re-verify from
    scratch: `POST api/v1/process/execution-template-categories/enabled`, `.../execution-template-types/enabled`,
    `.../execution-templates/by-category`, `POST api/v1/process-engine/execution/execution-templates/{id}/execute`,
    plus the Chat-Now resolution pair (`process/process-threads/by-id`, `ai-extension/app-processes/by-process`).
  - Real backend gap (already known, NOT this target's job to fix, just don't be surprised by it):
    `ExecutionTemplateCategory`/`ExecutionTemplateType`'s EF entities don't map `Code`/`IconUrl`/`Color`
    even though the DB columns exist — `CategoryList` degrades gracefully already; leave as-is.
- `BizFirstAiStudio\src\flow-studio\packages\process-pipelines-common\` — the OTHER real
  `ExecutionTemplate` consumer (`flow-pipelines` app's admin table view: search/create/edit/delete/
  execute a template). Its `ExecutionTemplate`/`ExecutionTemplateCategory` **types** and
  `ExecutionPipelinesApiClient.executeExecutionTemplate()`/`searchExecutionTemplates()` are a
  *second*, independently-built, differently-shaped client for the same backend domain. **Do not
  wire the new widgets through this package** — `ai-agent-catalog-ui` is the one built specifically
  for card/browse-and-run UI (per its own name and the ai-agent-browser-spec); process-pipelines-common
  is the admin/CRUD table tool. Only reference it if `ai-agent-catalog-ui` is ever found to be
  missing something process-pipelines-common already solved (e.g. cross-check wire-casing — both
  independently confirmed plain camelCase, not PascalCase, for this domain).
- `BizFirstAiStudio\src\hil\app\src\pages\AiAgentsPage.tsx` — WorkDesk's own "AI Agents" page,
  already just a few lines composing `AiAgentCatalogApiProvider` + `TemplateCatalog` — proof this
  package is meant to be reused verbatim by a new consumer, which is exactly what these two new
  App Studio widget types are.
- Chat-panel's reuse target, **not yet a package** (extraction is part of this target, see Phase 2):
  `BizFirstAiStudio\src\chatbots\apps\chatdesk\src\components\ChatWindow\ChatWindow.tsx` +
  `..\ChatWindowContainer\ChatWindowContainer.tsx` (+ `ConversationsPanel.tsx`,
  `..\services\processTriggerClient.ts`, `..\services\conversationResumeClient.ts`,
  `..\utils\toHilChatMessage.ts`). `ChatWindow` is presentation-only (renders via
  `@bizfirst/hil-ui-chat-window`'s real `HilChatTranscript`/`HilChatTextInput`, deliberately NOT
  that package's `HilChatWindow`/`HilChatComposer`, which are hard-wired to a live SignalR
  HilEngine — see `ChatWindow.tsx`'s own doc comment for why). `ChatWindowContainer` owns
  appID/processID/ConversationID/the conversations-list chrome and both send flows.
  - **Flow 1 (trigger a new conversation)**: real, wired — `ProcessTriggerClient.triggerNewConversation`
    POSTs `api/v1/process-engine/execution/execute`.
  - **Flow 2 (send a reply into an existing/resumed conversation)**: genuinely NOT working today —
    `resumeSession` is real state that is never populated (no SignalR listener, no server-side
    ConversationID→execution lookup exists yet); `ChatWindowContainer`'s own `disabledReason` logic
    already surfaces this honestly instead of pretending it works.
    **UPDATE 2026-08-25, explicit direction from Binoy: close this gap for real as part of this
    target ("yes, however use SRP") — do not just inherit the disabled state.** This is real,
    cross-repo work, not a small addition — see the new "Closing Flow 2" section below for the
    concrete plan and the two existing target specs it builds on
    (`flow-octopus-conversation-correlation.md`, `octopus-ai-agent-chat-resume.md`). The SRP
    constraint: build this as one dedicated, reusable resolver (backend endpoint + a thin frontend
    resolver hook), never as logic inlined into `ChatWindowContainer`/`chat-panel` — both the
    extracted chat-window package and ChatDesk itself must consume the same resolver, not two
    forks of "how do I turn a ConversationID into a resumable session."

## Closing ChatDesk's Flow 2 gap (new scope, added 2026-08-25 per Binoy — SRP required)

Two existing, already-designed target specs cover most of what's needed here — **read both before
planning this, do not re-derive their decisions**:

- `Documentation\Employees\agentic-coding\targets\flow-octopus-conversation-correlation.md` —
  decided design for a nullable `ConversationID` column on `Process_ProcessElementExecutions`
  (loose reference to `AIConv_Conversations.ConversationID`, Octopus stays unaware of Flow's
  schema — an explicit architectural constraint from Binoy, do not violate it). **Schema layer was
  DONE as of the memory this spec was written from; the application layer — `AiAgentNodeExecutor`
  actually writing this column when a chat node runs, and a reverse-lookup (given
  `ConversationID`, return `{ExecutionResID, NodeKey}`) — was NOT started.** Re-confirm current
  state during Phase 1 (Analyze) — this may have moved since.
- `Documentation\Employees\agentic-coding\targets\octopus-ai-agent-chat-resume.md` — the actual
  resume mechanism once you have `{executionResId, nodeKey}`:
  `OrchestrationProcessor.ExecuteContinuationAsync(executionResId, nodeKey, portKey, resumeData,
  ct)`, bypassing the dead Engage/`EngageSessionID` lookup path. Per prior-session memory this fix's
  code was "confirmed present" in a feature branch but never fully E2E-verified — re-confirm its
  real state during Phase 1 too, don't assume either "still broken" or "already works."

**What "closing the gap" concretely requires, SRP-scoped:**

1. **Backend, Flow side** (`BizFirstPayrollV3`): finish `flow-octopus-conversation-correlation.md`'s
   application layer — `AiAgentNodeExecutor` writes `ConversationID` onto the execution row — plus
   one new endpoint: given a `ConversationID`, resolve `{ExecutionResID, NodeKey}` (a thin
   read/lookup, not a new domain concept — follow this codebase's existing controller/service
   layering, don't invent a new pattern).
2. **Backend, Octopus side**: confirm/finish `octopus-ai-agent-chat-resume.md`'s
   `ExecuteContinuationAsync` resume path is real and callable from the reply endpoint the frontend
   already POSTs to (`api/hil/respond/...` per that spec) — this may already be done; verify, don't
   redo.
3. **Frontend — one new, dedicated resolver, not logic inlined into UI components.** A single
   small package/module (Phase 2 of this spec decides its exact name/location, following this
   codebase's existing "reusable capability = its own tiny package" convention — e.g.
   `ai-agent-catalog-ui`, `hil-ui-chat-window`) whose only job is: given a `ConversationID`, call
   the new backend lookup, then either (a) populate `ChatWindowContainer`'s `resumeSession` state
   with what it needs to submit a reply, or (b) surface a real, honest error if resolution fails —
   plus whatever live-update wiring (SignalR or polling — Phase 2 decides and documents why) keeps
   an open chat panel in sync with a suspended node's response. **This resolver must be consumed by
   both** the extracted chat-window package (`chatdesk-chat-window` or whatever Phase 2 names it)
   **and** `chat-panel`'s own composition of it — never two separate implementations of the same
   ConversationID→session lookup.
4. Once this resolver is real, `ChatWindowContainer`'s existing `disabledReason` logic for Flow 2
   should naturally stop triggering (the precondition it was checking is now satisfiable) — this is
   a removal of a now-unnecessary guard, not a parallel "Flow 2 v2" code path bolted on next to it.

**If Phase 1's re-confirmation finds this is bigger than fits inside this target's cycle budget**
(e.g. the backend pieces alone are substantial, multi-repo, security-sensitive work) — say so
explicitly in the Plan output rather than silently shipping a half-working resolver. It is
legitimate for the Planner to recommend running `flow-octopus-conversation-correlation.md`'s
application layer as its own separate agentic-coding-loop pass first (it already has 90% of its own
target spec written), then coming back to this target once that's landed — flag that recommendation
back to the calling session rather than guessing which way to go.

## Widget-handler contract (read the reference implementation before writing a new one)

`widget-handlers-form-widget` (`FormWidgetConfig.ts`/`FormWidgetHandler.ts`/`FormWidgetRenderer.tsx`)
and `widget-handlers-content-widget` (`ContentWidgetConfig.ts`/`ContentWidgetHandler.ts`/
`ContentWidgetRenderer.tsx`, `ContentSanitizer.ts`) are the two existing implementations of:

- `IWidgetHandler` (`widget-handlers-core\src\IWidgetHandler.ts`): `widgetType: string`,
  `load()`/`unload()` (async lifecycle, usually no-ops — see `BaseWidgetLoadHandler`), `render(ctx):
  WidgetRenderResult` (pure data resolution, no DOM), optional `Component` (the React mount point
  for the result).
- **`WidgetRenderResult` is a closed discriminated union** in that same file:
  `{type:'form'|'content'|'list'|'error', ...}`. **Adding new widget types requires adding new
  members to this union** (e.g. `{type:'workflow-template', ...}`, `{type:'workflow-template-list',
  ...}`, plus reusing `{type:'error', message}` for both) — this is a shared, cross-cutting edit to
  a foundational package every widget type depends on, do it once, carefully, not per-widget-type.
- **`WidgetType` is a closed string union** in `app-handlers-core\src\types\WidgetRecord.ts`
  (`export type WidgetType = 'form' | 'content';`) — must become `'form' | 'content' |
  'workflow-template' | 'workflow-template-category' | 'chat-panel'`. Same "shared core, edit once"
  note.
- `BaseWidgetLoadHandler.resolveConfig<T>(widget, defaults)` — shallow-merges `widget.configuration`
  over per-handler defaults. Follow this pattern for the three new config shapes (don't hand-roll
  merging).
- Registration points (every one of these needs the three new handlers added, confirmed current
  this session — re-confirm at Plan time, code may have moved):
  1. `app-studio-designer-components-react\src\preview\useLivePreviewEngine.ts` — the studio's own
     live-preview canvas registry (`registry.register('form', new FormWidgetHandler()); ...`).
  2. `app-player\src\App.tsx` — `getWidgetRegistry()` (this session added `form`'s registration
     here; before that it was missing entirely — don't assume app-player's registry is a stale
     mirror of #1, re-read both).
  3. `app-player\vite.config.ts` and `app-studio-designer\vite.config.ts` — each new
     `widget-handlers-<type>-widget` package needs a `path.resolve` alias in **both** files (same
     pattern as `@app-studio/widget-form` this session), plus any *new* cross-package dependency
     the new handlers pull in (e.g. `ai-agent-catalog-ui`, `hil-ui-chat-window`, the extracted chat
     package) needs its own alias in both, mirroring how this session added the full Atlas Forms
     alias block to both files for the `form` type's transitive deps. Check each new package's
     actual `import` graph before assuming which aliases are needed — don't guess from a similar
     package's list.
  4. `AddWidgetModal.tsx` (`app-studio-designer-components-react\src\modals\AddWidgetModal.tsx`) —
     currently a hardcoded `'form' | 'content'` radio pair; see Phase 2/3 below for making this
     data-driven.
  5. Both apps' `package.json` — new workspace deps for whatever's newly aliased (see this
     session's `@atlas-forms/client-js` addition to `app-studio-designer/package.json` as the
     precedent — minimal, only what's directly imported, not every transitive alias).

## Files known to be in scope (confirm during planning, code may have moved)

- `app-studio\packages\widget-handlers-core\src\IWidgetHandler.ts` (WidgetRenderResult union)
- `app-studio\packages\app-handlers-core\src\types\WidgetRecord.ts` (WidgetType union)
- New packages to create, one per widget type (SRP — mirror `widget-handlers-form-widget`'s
  3-4-file shape exactly): `app-studio\packages\widget-handlers-workflow-template-widget\`,
  `app-studio\packages\widget-handlers-workflow-template-category-widget\`,
  `app-studio\packages\widget-handlers-chat-panel-widget\`.
- New package to create (Phase 2, chat-panel's prerequisite):
  `bizfirst-common\chatdesk-chat-window\` (or similar — name it to match `ai-agent-catalog-ui`'s
  precedent of living in `bizfirst-common` "because I will be reusing this in many apps", per that
  package's own dev log quoting Binoy) — the extracted `ChatWindow`/`ChatWindowContainer` +
  services. After extraction, `chatbots\apps\chatdesk\src\` must be updated to consume the new
  package instead of its own local copies (a real app, not just the new widget, depends on this
  working correctly — don't leave ChatDesk on a stale fork of the code).
- `app-studio\packages\app-studio-designer-components-react\src\preview\useLivePreviewEngine.ts`
- `app-studio\apps\app-player\src\App.tsx`, `app-studio\apps\app-player\vite.config.ts`,
  `app-studio\apps\app-player\package.json`
- `app-studio\apps\app-studio-designer\vite.config.ts`, `app-studio\apps\app-studio-designer\package.json`
- `app-studio\packages\app-studio-designer-components-react\src\modals\AddWidgetModal.tsx`
- New small file for the alias registry (Phase 2 decides exact location/shape — likely
  `app-studio\packages\app-handlers-core\src\types\WidgetTypeRegistry.ts` or a new tiny package if
  Phase 2's design finds a good reason to keep it out of core — default to core unless there's one).
- `bizfirst-common\ai-agent-catalog-ui\src\api\aiAgentCatalogApiClient.ts`,
  `..\src\hooks\` (new `useExecutionTemplateById.ts` — Real gap #1) — this package is a dependency
  the new widgets consume, but adding one missing method/hook to it is small, in-scope, and much
  better than the new widget package reimplementing template-by-id fetching itself.

## Real gaps found during research — flag explicitly, don't silently paper over

1. **No single-template-by-ID fetch exists in `ai-agent-catalog-ui` yet.** The *backend* endpoint
   already exists (`BaseExecutionTemplateController.GetById`, `[FromBody] GetByIdWebRequest`, route
   prefix `api/v1/process/execution-templates`, action confirmed present at
   `BizFirstPayrollV3\src\mvc-server\Ai\Process\BizFirst.Ai.Process.Api.Base\Controllers\BaseExecutionTemplateController.cs`)
   — only the frontend client method + hook are missing. Add `getTemplateById(executionTemplateID)`
   to `AiAgentCatalogApiClient` (mirror `getTemplatesByCategory`'s shape) and a
   `useExecutionTemplateById` hook (mirror `useExecutionTemplatesByCategory`'s shape) as part of
   Phase 3. Confirm the actual wire route (`.../by-id`, camelCase request body) by reading the
   controller action directly, not by assuming it matches the `by-category` action's route shape.
2. **ChatWindow/ChatWindowContainer are not a package today** — see "Prior analysis" above.
   Extraction is real, in-scope work (Phase 2/3), including updating ChatDesk itself to consume the
   new package so there's exactly one copy, not two forks.
3. **ChatDesk's Flow 2 (send a reply into a resumed conversation) is genuinely broken/disabled
   upstream — UPDATE 2026-08-25: Binoy has explicitly asked for this to be closed as part of this
   target, with SRP.** See the dedicated "Closing ChatDesk's Flow 2 gap" section above for the
   concrete, cross-repo plan. Do not silently fall back to "inherit the disabled state" — that was
   this spec's original default before this direction was given.
4. **No `AiAgentCatalogApiProvider` is currently mounted in either `app-player` or
   app-studio-designer's live-preview tree.** `workflow-template`/`workflow-template-category`'s
   handlers need a way to reach a configured `AiAgentCatalogApiClient` — Phase 3's Coder must
   decide (and document the decision, don't just wing it) whether to mount the provider at the
   `AppPlayer`/live-preview-engine level (once, shared) or have each handler's `Component`
   construct its own client instance the way `FormWidgetRenderer.tsx` does for
   `formDefinitionApiClient`/`formDataApiClient` (module-level singletons off `AtlasFormsClient`).
   Prefer whichever keeps the widget package itself simplest and doesn't require every future
   widget-consuming app to remember to wrap a provider — read how `formDefinitionApiClient`/
   `formDataApiClient` get their config today (`@atlas-forms/api-client-js`) as the closer
   precedent, since that's also "a widget handler needs a configured API client, App Studio has no
   central provider for it."
5. **`chatDeskBaseUrl`** — `TemplateCard`'s Chat-Now button and `chat-panel`'s own conversation
   trigger both need ChatDesk's real base URL. `AiAgentsPage.tsx` (WorkDesk) already has the
   precedent: `VITE_CHATDESK_APP_URL` env override, default `http://localhost:6111`, with a comment
   noting no shared `@passport/app-urls` entry exists for ChatDesk yet. Follow the same pattern in
   both new apps' env config rather than hardcoding the URL differently in a third place.

## Alias/display-name design (Phase 2 — design this concretely, don't leave it vague)

Binoy's concrete example: the `workflow-template` widget type (backed by the `ExecutionTemplate`
entity) should display as **"Workflow Agent"** to end users — not the raw `workflowType`/entity
name. Design and implement:

- A small, data-driven registry — one entry per registered widget type — of `{widgetType: string,
  label: string, description?: string, icon?: ...}`. Exact shape and location are Phase 2's call;
  candidates: alongside `WidgetType` in `app-handlers-core` (keeps the type and its display name
  next to each other), or a new tiny package if there's a real reason core shouldn't own display
  strings (e.g. i18n concerns) — default to co-locating with `WidgetType` unless Phase 2 finds a
  concrete reason not to.
- `AddWidgetModal.tsx`'s "New Widget" tab must iterate this registry to render its type picker
  instead of the current hardcoded two-row radio group — a real refactor of that component, not
  just new rows appended to the hardcoded pair (5 types today, more later; hardcoding doesn't scale
  and Binoy explicitly asked for the alias system specifically to avoid this).
- Concrete required aliases for this target: `form` → "Form Widget" (existing, unchanged),
  `content` → "Content Widget" (existing, unchanged), `workflow-template` → **"Workflow Agent"**,
  `workflow-template-category` → (Phase 2 to propose and confirm with Binoy — e.g. "Workflow Agent
  Category" or "Workflow Agent Group"; don't guess silently, this is exactly the kind of naming
  Binoy cares about and asked to control), `chat-panel` → (same — propose and confirm, e.g. "Chat
  Window").

## Config shapes to design in Phase 2 (concrete, not vague)

- `WorkflowTemplateWidgetConfig`: `{ executionTemplateID: number }` at minimum — check whether
  `TemplateCard`'s props need anything else host-configurable (e.g. show/hide description, a
  `chatDeskBaseUrl` override) and whether that should be per-widget config or an app-wide App
  Studio setting; default to per-widget config unless Phase 2 finds a reason it must be app-wide.
- `WorkflowTemplateCategoryWidgetConfig`: `{ executionTemplateCategoryID: number }` — same
  question about a description/heading toggle above the card grid (`TemplateCatalog`'s header
  shows `selectedCategory.name`/`.description`; decide whether `workflow-template-category`'s
  renderer shows an equivalent heading or is card-grid-only).
- `ChatPanelWidgetConfig`: needs `appID`+`processID` (what `ChatWindowContainer` requires today) —
  but an App Studio widget config is more naturally expressed as "which workflow template does
  this chat panel talk to" (`executionTemplateID`, matching `chat-panel`'s sibling widgets'
  config shape) with `appID`/`processID` resolved server-side/at-runtime the same way
  `TemplateCard`'s own Chat-Now button does today (`client.resolveChatTarget(executionScope,
  scopeReferenceID)`) rather than asking the App Studio app author to already know ChatDesk's
  internal `appID`/`processID` numbers. Phase 2 must decide and document which of these two shapes
  is used — don't leave both half-implemented.

## Widget-config ID fields need a real lookup UI, not a raw number input (added 2026-08-25, Binoy)

Same lesson as this session's `FormLookupField` fix for the `form` widget type (`AddWidgetModal.tsx`
used to have a raw `<input type="number">` for `formID` — real, live-reproduced bug: nobody could
know a `FormID` by heart, and it shipped broken because the config-key casing didn't even match
what the handler read). **Do not repeat that pattern for `executionTemplateID` /
`executionTemplateCategoryID`.** Both need a real search-as-you-type lookup control in
`AddWidgetModal.tsx`'s (now data-driven, see Phase 2/3) widget-creation form — same UX shape as
`FormLookupField`: type to filter, shows `{ID} — {name}`, select to set the config value.

- Check whether `ai-agent-catalog-ui` already exposes a general "list/search templates" capability
  beyond `useExecutionTemplatesByCategory` (which is scoped to one category) — `TemplateCatalog`
  itself must resolve categories/templates somehow; read its real implementation before assuming
  one does or doesn't exist.
- If no free-text/global search-across-all-templates method exists yet, add a minimal one to
  `AiAgentCatalogApiClient` (mirror the `getTemplateById`/`useExecutionTemplateById` addition
  already planned for Real gap #1 — same "small, safe, additive" bar, don't build a new subsystem
  for this).
- For `executionTemplateCategoryID`, the equivalent lookup is over categories, not templates —
  confirm whether `ai-agent-catalog-ui` already has a categories-list method/hook (likely, since
  `TemplateCatalog` needs to show a category list somewhere) before adding a new one.
- Reuse `FormLookupField` itself (`@atlas-forms/ui-components-react`) as the UI component if its
  generic `fetchForms`/`loadForm`-shaped props (rename-agnostic — it's really "fetch items"/"load
  one item by id") fit cleanly for this data source too, rather than building a second bespoke
  lookup component that duplicates the same search-dropdown mechanics
  (`useLookupDropdown`/debounce/click-outside). Only build a new component if `FormLookupField`'s
  shape genuinely doesn't fit (e.g. it's too Form-specific in ways that can't be generalized
  without contorting it) — judge and document that decision, don't default to duplicating without
  checking reuse first.

This applies to **both** `workflow-template` (Phase 3) and `workflow-template-category` (Phase 4)
— add it to both phases' acceptance criteria, not just one.

## Environment / build notes (frontend-only target — no backend build step)

This is a pure frontend target (React/TypeScript, pnpm workspaces under `BizFirstAiStudio`) except
for the one confirmed-safe backend addition in Real gap #1's own scope (a new frontend client
method/hook calling an *existing* backend action — no new controller action, no C# changes). If
Phase 1/3 discovers a genuine need for a new backend endpoint anywhere else in this target, stop
and flag it back rather than silently adding one — this target was scoped and researched as
frontend-only.

- Dev servers involved for E2E verification: `app-studio-designer` (port 6109),
  `app-player` (port 6130), `chatbots\apps\chatdesk` (port 6111, per `AiAgentsPage.tsx`'s
  documented default), and the Consolidated WebApi (port 10001) for real data. This session
  already has working `.env.development`/vite-alias patterns for both `app-studio-designer` and
  `app-player` (added this session, fixing real bugs in both) — read those two `vite.config.ts`
  files as the current, correct baseline before adding more aliases, not the state before this
  session's fixes.
- This dev machine is RAM-constrained (~7.7GB total, confirmed repeatedly this session) — running
  more than 2-3 vite dev servers plus Chrome plus a build has caused real slowdowns/frozen tabs
  this session. Stagger dev-server startup during E2E verification; don't run all of
  app-studio-designer + app-player + chatdesk + flow-studio + WorkDesk simultaneously unless
  actually needed for a specific cross-app check.
- Login/auth: same SSO handoff-code flow as every other app in this monorepo
  (`@bizfirst/common-auth-react`'s `SsoAuthGate`) — login app at port 8001, credentials known to
  this session (ask the calling session if needed, don't guess/hardcode credentials into this
  spec).

## Acceptance criteria — 5 phases, "analyze, design, create one by one" per Binoy's explicit request

### Phase 1 — Analyze (confirm, don't re-derive)

1. Re-confirm every real file path and registration point listed above still matches current code
   (code may have moved since this spec was written) — explicitly note any drift found.
2. Re-confirm the exact wire route/request shape for `BaseExecutionTemplateController.GetById`
   (Real gap #1) by reading the controller action directly.
3. Re-confirm `ChatWindow.tsx`/`ChatWindowContainer.tsx`'s current file contents match what's
   quoted in this spec (Flow 1 wired, Flow 2 disabled) — if someone has since fixed Flow 2,
   `chat-panel` should NOT re-disable a now-working feature; verify current state, don't assume
   this spec's snapshot is still accurate.
5. Re-confirm the current real state of both `flow-octopus-conversation-correlation.md`'s
   application layer and `octopus-ai-agent-chat-resume.md`'s resume path (see "Closing ChatDesk's
   Flow 2 gap" above) — both may have moved since this spec's research pass. Report the finding
   explicitly in the Plan output, including a recommendation if the scope should be split across
   two loop runs.
4. Confirm whether `AiAgentCatalogApiProvider` is genuinely absent from both `app-player` and
   app-studio-designer's live-preview tree (Real gap #4) — if it's since been added for some other
   reason, reuse it rather than re-deciding.

### Phase 2 — Design (write it down concretely before touching code)

1. The widget-type alias/display-name registry: exact shape, exact file location, and
   `AddWidgetModal.tsx`'s data-driven refactor plan.
2. The three widget config shapes (see "Config shapes to design" above) — resolved, not left as
   open options.
3. Where `AiAgentCatalogApiClient`/`AiAgentCatalogApiProvider` gets configured/mounted for the two
   workflow widgets (Real gap #4) — resolved, not left as open options.
4. The `chatdesk-chat-window` (or equivalently named) extraction plan: exact new package location,
   exact files moved, exact ChatDesk-app updates needed to consume it afterward.
5. `chat-panel`'s exact config shape (Real gap resolution from "Config shapes" above).
6. The Flow-2 resolver's exact name/location/shape (see "Closing ChatDesk's Flow 2 gap" above) and
   which SignalR-vs-polling live-update approach it uses, decided and documented, not left open.
7. Whichever V1 scope statement ends up true after Phase 1's re-confirmation (Flow 2 working, or —
   only if Phase 1 explicitly recommended splitting the backend work into its own loop run first —
   still honestly disabled for this run) written into the package's own doc
   comment/DevelopmentHistoryLog.md, mirroring how ChatWindowContainer.tsx documents its scope
   today.

### Phase 3 — Create `workflow-template`

1. `widget-handlers-workflow-template-widget` package exists, 3-4 files mirroring
   `widget-handlers-form-widget`'s shape (Config/Handler/Renderer + any needed service file).
2. `WidgetRenderResult`/`WidgetType` unions extended (shared core edit, done once, both new types'
   members added even though `workflow-template-category` isn't built until Phase 4 — avoids a
   second core-package edit).
3. `getTemplateById`/`useExecutionTemplateById` added to `ai-agent-catalog-ui` (Real gap #1).
4. Registered in `useLivePreviewEngine.ts` and `app-player/src/App.tsx`'s registry; aliased in both
   apps' `vite.config.ts` (+ `package.json` deps for anything newly imported directly).
5. `AddWidgetModal.tsx` updated to the data-driven picker (Phase 2's design), showing "Workflow
   Agent" as this type's label, and lets a user actually create a `workflow-template` widget bound
   to a real `ExecutionTemplateID`.
6. **Live-verified** (not just "builds clean"): create an app in App Studio, add a
   `workflow-template` widget bound to a real, existing `ExecutionTemplateID`, see the real
   `TemplateCard` render in both the studio's live-preview canvas AND in `app-player`'s standalone
   preview (mirroring how this session verified the `form` widget type in both places), and
   confirm clicking Execute (or Chat Now, if the chosen test template is conversational) actually
   does something real (a new execution starts, or ChatDesk opens) — not just that the card
   renders.

### Phase 4 — Create `workflow-template-category`

1. `widget-handlers-workflow-template-category-widget` package exists, reusing `TemplateCard` (via
   `ai-agent-catalog-ui`, not a copy) for each card in the grid — no duplicated card-rendering code
   anywhere in this new package.
2. Same registration/alias/AddWidgetModal wiring as Phase 3, for this second type.
3. **Live-verified**: bind a widget to a real `ExecutionTemplateCategoryID` with more than one
   template in it, confirm all templates in that category render as cards in both the studio
   canvas and `app-player`, and confirm at least one card's action (Execute or Chat Now) works.

### Phase 5 — Create `chat-panel`

1. `chatdesk-chat-window` (or Phase 2's chosen name) package exists under `bizfirst-common`,
   containing the extracted `ChatWindow`/`ChatWindowContainer`/`ConversationsPanel`/
   `processTriggerClient`/`conversationResumeClient`/`toHilChatMessage` (exact file list per Phase
   2's plan). `chatbots\apps\chatdesk` updated to import from the new package instead of its own
   local copies — **ChatDesk itself must still work identically after this refactor** (build it,
   confirm no behavior change, this is an extraction not a rewrite).
2. `widget-handlers-chat-panel-widget` package exists, thin — composes the extracted
   `ChatWindowContainer` the same way `AiAgentsPage.tsx` composes `TemplateCatalog`.
3. Same registration/alias/AddWidgetModal wiring as Phases 3/4, showing Phase 2's chosen "chat
   panel" label.
4. The Flow-2 resolver from "Closing ChatDesk's Flow 2 gap" above is built and wired into the
   extracted `ChatWindowContainer` (consumed by both it and `chat-panel`'s composition of it) —
   unless Phase 1 explicitly recommended deferring the backend portion to a separate loop run, in
   which case this criterion becomes "honestly disabled, same as ChatDesk today" and that deferral
   must be stated plainly in the run's final report, not silently substituted.
5. **Live-verified**: add a `chat-panel` widget to an App Studio app, confirm it renders a real
   chat window, confirm sending a first message actually triggers a real process execution (Flow
   1), AND — if the Flow-2 resolver was built this run — confirm replying into that same
   conversation actually resumes the suspended node for real (not just that the UI stops showing
   "disabled"). If Flow 2 was deferred per item 4's fallback, confirm the UI still honestly shows it
   as unavailable rather than silently failing.

## Live test app for E2E verification

Use App Studio Designer directly (`app-studio-designer`, port 6109) + `app-player` (port 6130) —
create a real test app (e.g. "Widget Types Demo") with one of each new widget type, save it, and
verify both the studio's live-preview and `app-player`'s standalone render for every phase's
acceptance criteria, exactly as this session did for the `form` widget type this same run (see
this session's own verification of FormID 5008 in both surfaces as the concrete precedent for what
"live-verified" means here — screenshots/DOM checks via claude-in-chrome, not just "the build
succeeded").
