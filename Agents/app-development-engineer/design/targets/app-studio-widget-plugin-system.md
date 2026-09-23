# Target: App Studio — "Bring Your Own Widget Type" plugin system (research + phased proposal)

**Status: research/design proposal only — no code changed.** Written 2026-09-15 at Binoy's request,
to evaluate building a widget-type plugin system for App Studio as a foundation for reusable
use-case widget families (blog, help docs, FAQ, testimonials, portfolio, events, jobs, pricing,
team, resources, changelog, forum, catalog, newsletter archive, podcast/video series, glossary).

## Problem statement

Binoy wants to know whether App Studio can move from "a fixed, compiled-in catalog of 18 widget
types" to "a widget type can be registered externally/dynamically" — and, if so, how far that can
realistically go: from BizFirst's own team building faster, up to genuine third-party plugins.

**Headline finding, before anything else**: this is not a green-field question. Binoy has *already*
commissioned exactly this initiative on the backend, in his own words, captured verbatim in a
migration comment written before this research started:

> "I want to start approaching widgettypes as pluggable components and anyone can build them and
> bring in later" — `AppStudio\migrations\V017_UP_AIExt_WidgetTypes_Create.sql`

That migration (applied live) replaced the old hardcoded C# array + DB `CHECK` constraint with a
real, row-driven `AIExt_WidgetTypes` lookup table. **The backend half of "pluggable widget types" is
already built and live.** What is NOT built, anywhere in the frontend, is any mechanism to make a
new type actually *render* without a code change and a rebuild. That gap — not the backend — is the
real subject of this proposal.

---

## Part 1 — The full current widget-type resolution path, traced end to end

### 1.1 Backend: three independent gates, only one of which is genuinely data-driven

1. **`AIExt_WidgetTypes`** (`BizFirstFiDB\BizFirstFiV3DB\...\dbo\Tables\AIExt_WidgetTypes.sql`,
   entity `WidgetTypeDefinition : BaseEntity`,
   `BizFirst.Ai.AIExtension.Domain\Entities\WidgetTypeDefinition.cs`) — `WidgetTypeID`, `WidgetType`
   (string key), `Name`, `Description`, `Enabled` (bool), plus the standard `BaseEntity` audit/tenancy
   columns. **Confirmed genuinely data-driven and live**:
   `WidgetService.ValidateAsync` (`BizFirst.Ai.AIExtension.Service\Services\WidgetService.cs:140-156`)
   calls `_widgetTypeRepository.IsValidAndEnabledAsync(entity.WidgetType, ...)` — a real DB query, not
   a hardcoded list — and throws `ArgumentException` only if no non-deleted, `Enabled=1` row matches.
   **A new `WidgetType` string is usable by the real `Widget`/`AppWidget` create/update pipeline today
   by inserting one row into `AIExt_WidgetTypes` — zero code change, zero redeploy.**
   `IWidgetTypeRepository` inherits full generic CRUD from `IRepository<WidgetTypeDefinition,int>`, so
   the plumbing to manage this catalogue programmatically already exists (no dedicated
   `WidgetTypesController` was found exposing it over HTTP as of this research pass — that would be a
   small, cheap addition, not new architecture).
   `IsValidAndEnabledAsync`'s own doc comment: intentionally **not tenant-scoped** — "widget types are
   a shared, platform-wide registry of pluggable capabilities... not tenant-owned data," mirroring
   `Template_DataTemplateTypes`' role for `Template_DataTemplate`. This is the right existing pattern
   to build on.

2. **`BizFirst.Ai.Mcp.Tools.AppStudio\Tools\WidgetTypeCatalog.cs`** — a completely separate, hardcoded
   C# `Dictionary<string,string>` of the same 18 types, used only by the MCP tool layer
   (`create_widget`/`update_widget_placement` argument validation for AI-agent-driven builds). **This
   is now stale relative to #1** — it has no awareness of `AIExt_WidgetTypes` at all, so a type added
   via the real catalogue today would still be rejected by any MCP-driven `create_widget` call
   ("Unknown widgetType '...' Must be one of: ..."), and conversely this file's own doc comment admits
   its structural/shape validation only covers 9 of the 18 types.

3. **Per-widget-type `Configuration` JSON shape** — **not generalized anywhere**. Beyond `ISJSON()` at
   the DB-constraint level (any valid JSON passes), there is no shared config-schema mechanism. Shape
   checking is ad hoc and incomplete: `WidgetTypeCatalog.cs`'s `RequiredConfigField` dictionary
   hand-checks one required field for 9/18 types (MCP path only); the real `WidgetService.ValidateAsync`
   (item #1, the actual enforcement path for Designer-driven creates) checks **only** that `WidgetType`
   is a known/enabled string — it does **not** validate `Configuration`'s shape against anything at
   all. Every widget type's real config contract lives only as a TypeScript interface
   (`*WidgetConfig.ts`) inside its own frontend package, consumed at render time, never checked
   server-side. **This directly answers investigation item #3**: the `content` widget's
   `allowScripts`/DOMPurify sanitization pattern is specific to that one widget type
   (`ContentSanitizer.ts`) and does not generalize — there is no reusable "define a config schema,
   get validation for free" mechanism today. Any new widget type, first-party or external, gets
   exactly as much backend shape-validation as someone deliberately writes for it.

**Backend summary**: the *allow-list* problem (is this WidgetType string permitted at all) is
solved, live, and genuinely pluggable — but two of the three gates that actually run in production
(`WidgetTypeCatalog.cs` for MCP, and the total absence of `Configuration`-shape validation) have not
caught up to it, and a config-schema contract does not exist as reusable infrastructure.

### 1.2 Frontend: a closed compile-time type, a real registry pattern, zero dynamic loading

1. **`WidgetType`** (`app-handlers-core\src\types\WidgetRecord.ts:94-112`) is a **TypeScript literal
   union of exactly 18 strings**. This is a compile-time closed set — a 19th value is a type error
   everywhere this type is used, full stop. Extending it requires editing this file and rebuilding
   every package that imports `@app-studio/core`.

2. **`WIDGET_TYPE_REGISTRY`** (`app-handlers-core\src\types\WidgetTypeRegistry.ts`) — a hardcoded
   array of `{widgetType, label, description}`, iterated by `AddWidgetModal`'s "New Widget" tab to
   populate the creation picker. Its own doc comment is explicit about the scope split: "this file
   only controls what the creation UI *offers*, not what actually renders."

3. **What actually renders — `WidgetRegistry`** (`widget-handlers-generic\src\WidgetRegistry.ts`): a
   genuinely well-shaped `Map<WidgetType, IWidgetHandler>` with `register`/`get`/`getOrThrow`/`has`/
   `unregister`/`registeredTypes()`. **This part of the architecture is already a real registry
   pattern, not a `switch` statement** — a meaningful, reusable piece of plumbing.
   **But it is populated exclusively via static ES `import` statements of each
   `@app-studio/widget-{type}` npm package, in exactly three bootstrap sites**, all with the identical
   shape (18, or a static-export subset of 11, hardcoded `.register(type, new Handler())` calls):
   - `apps\app-player\src\App.tsx` (the standalone App Player runtime, all 18 types)
   - `apps\static-app-player\src\services\widgetRegistry.ts` (static-export player, 11
     "static-eligible" types only — `form`/`hil-inbox`/`signin`/`notifications`/`workflow-template`/
     `workflow-template-category`/`chat-panel` deliberately excluded, matching
     `StaticAppExportService.BlockedWidgetTypes`)
   - `app-studio-designer-components-react\src\preview\useLivePreviewEngine.ts` (the Designer's own
     live-preview canvas, all 18 types)

   **A new widget type today requires: a new npm package in the monorepo, a new `import` + `.register()`
   call added to some or all of these three files (registration-parity is a real, already-flagged
   risk — `app-studio-pages-navigation.md`'s Phase 4 calls out exactly this "keep both in sync
   deliberately" concern for a different feature), and a full rebuild + redeploy of every affected
   frontend app.** There is no way around this today.

4. **No dynamic/runtime loading mechanism exists anywhere in this codebase.** Searched the entire
   `app-studio` frontend tree for `React.lazy`, dynamic `import()`, Module Federation config, and any
   remote-component-loading pattern. Result: **zero genuine matches.** The only real dynamic
   `import()` calls found (`WidgetEditorFields.tsx`, three call sites: `@bizfirst/expressions-designer`,
   `../htmlEditor/HtmlEditorLauncher`, `@atlas-forms/form-editor-react`) are ordinary Vite
   code-splitting of **known, first-party, monorepo-bundled** packages — lazy-loaded for bundle-size
   reasons, resolved entirely at build time. This is meaningfully different from fetching and
   executing a bundle from an unknown, runtime-supplied URL. No Module Federation plugin, no
   `remoteEntry.js` pattern, no iframe/postMessage sandboxing boundary, no "vetted bundle fetched
   from a plugin registry and eval'd" pattern exists anywhere in this repo.

**This is the single most important fact for scoping the rest of this proposal**: option (c) —
true runtime plugin loading — is not "extend an existing mechanism a bit further." It is "build a
second rendering pathway with sandboxing decisions this codebase has never had to make," exactly as
the task brief anticipated should be the case if no dynamic-loading mechanism turned up. It did not
turn up.

---

## Part 2 — The three ambition tiers, assessed against what's real today

### (a) First-party modular — new types as new monorepo packages, same static registry

**This is what the codebase already does for every one of its 18 types.** Genuinely cheap, in the
sense that the pattern, conventions, and tooling already exist end to end: a new
`widget-handlers-{type}-widget` package (config interface + handler + renderer), a row in
`WidgetTypeRegistry.ts`, a row in `AIExt_WidgetTypes` (now real, no redeploy needed for the allow-list
half), and three `.register()` call-sites to touch. Not "pluggable" in any sense beyond "a new
first-party package, well organized." Zero new architecture required. **This is the floor, not a
proposal — it's already how the team works.**

### (b) Config-driven "widget kits" — a declarative way to define a widget's shape/rendering

**The closest thing to this today is the Content Widget Template mechanism** (`IsTemplate=1`
`content`-type widgets, clone-on-place via `createWidgetAndPlace`, documented in
`content-widget-templates.md`) — 34 real, seeded, `--app-var-*`-themed HTML/CSS templates
(Timeline, Testimonial, Pricing Table, Feature Grid, plus 30 more) that a builder (human or agent)
clones and edits rather than hand-authoring from scratch. **This already proves the core idea works
for one widget type** (`content`) but is not a generalized "define a new widget shape without
writing React" system — it's templated HTML strings inside one fixed widget type's `content` field,
with no data-binding, no list/detail semantics, no structural fields beyond `content`/`format`/
`allowScripts`.

A real Phase-1-scoped "widget kit" mechanism would need to be a genuinely new, bounded,
declarative layer — e.g., a JSON schema for a config shape + a small set of pre-built, parameterized
rendering primitives (a list-of-records block, a detail-record block, a card-grid block) that a
kit's JSON selects and configures, rather than arbitrary HTML/CSS. This is real, scoped design work,
not zero-cost — but it is bounded-expressiveness, no-arbitrary-code, and buildable entirely with
patterns already proven in this codebase (JSON `Configuration`, the theming token contract, the
`AppPage`/`entityType`/`dataID` routing engine — see Part 4).

### (c) True dynamic/runtime plugin loading

Per Part 1.2, this has **no existing foundation to extend**. A serious, honest accounting of what it
requires, none of which exists today:

- A **sandboxing boundary** for third-party render code — the realistic choices are an iframe +
  `postMessage` contract (safest, most isolation, worst DX/performance — no shared React tree,
  no easy access to the theme cascade since `--app-var-*` custom properties don't cross an iframe
  boundary without explicit relay), or a Web Component boundary with a constrained API surface
  (better DX, weaker isolation, requires real Shadow DOM + CSP discipline), or (Shopify's actual
  model, see Part 3) *no client-side third-party code execution at all* — the "plugin" only ever runs
  server-side/at-publish-time, emitting data the host renders with host-owned components.
- A **bundle-fetch-and-execute pipeline** — versioning, integrity verification (subresource
  integrity or an equivalent signing scheme), a CSP that still permits it (this codebase's own
  `content` widget sanitization work this week — DOMPurify silently stripping `<style>` — is a small
  taste of how much can go subtly wrong the moment untrusted content meets a security boundary; a
  full third-party JS bundle is a categorically bigger version of the same problem).
- A **compatibility/versioning contract** for the plugin API surface (props shape, theme token
  access, action/event system) that can evolve without breaking every published plugin — this
  codebase has no precedent for maintaining a stable external API surface across releases.
- A **security review and a vetting/review-before-publish process** before this could be offered to
  actual external third parties, not just BizFirst's own team — code review, a marketplace/publish
  gate, a takedown mechanism, and ongoing monitoring for a vector that runs in every visitor's
  browser session on every app using that plugin (XSS blast radius = every tenant/app that adopted
  it, not just the plugin author's own data).

None of this should be read as "don't ever do it" — it should be read as "this is its own initiative,
with its own design doc, its own security review, and a real cost," not a checkbox inside Phase 1.

---

## Part 3 — Industry grounding, checked against Part 1's finding

- **WordPress**: real third-party plugin ecosystem, but the reason it works is structural, not just
  policy — PHP plugins execute **server-side**, in a shared-fate single-tenant-per-install model (one
  WordPress site = one process, one plugin author's code affects that site owner only, not a
  multi-tenant platform's other customers). BizFirst is multi-tenant SaaS; a WordPress-style
  "any code runs" model would mean one tenant's installed plugin executing in the same runtime other
  tenants' apps render through — categorically different, worse blast radius. WordPress's own
  security track record (a constant stream of plugin-sourced vulnerabilities) is the visible cost of
  this model even in its single-tenant-friendlier context.
- **Webflow**: **no true third-party widget/element plugin system**, despite being a mature,
  well-funded page builder — custom code is embed-only (an `<iframe>`/raw HTML/JS embed block a
  designer manually pastes in, not a registered, catalog-listed component type). This is the closest
  precedent to App Studio's own `content` widget's `allowScripts` escape hatch. Telling, per the task
  brief's framing: a serious competitor with every incentive to build a plugin marketplace has not,
  which is real evidence that "constrained embed, not a registered extensible type system" is a
  reasonable, defensible product position, not merely a limitation.
- **Shopify**: the actual sanctioned model for what BizFirst is asking about. Theme App
  Extensions and App Blocks are the mechanism closest to "install a third-party widget type" —
  crucially, the extension's UI is either (a) rendered through **host-owned, sandboxed injection
  points** the theme explicitly declares (App Blocks — a merchant places a pre-approved block into a
  pre-defined slot; the block's own logic runs through Shopify's own controlled surface, not
  arbitrary injected script) or (b) served through **App Bridge inside an iframe** for the admin-side
  app surface, with a real app review process gating the public App Store. This is a real, working
  precedent for option (c) done safely — and it is a materially bigger engineering + policy
  investment than anything in App Studio today, matching this proposal's Part 2(c) cost estimate,
  not undercutting it.
- **Notion**: embeds are iframe-sandboxed third-party URLs (a Figma/Loom/etc. embed), not a
  registered "widget type" with access to Notion's own data model or theme — the same
  iframe-isolation-over-integration trade-off Webflow and a cautious version of Shopify both make.

**Synthesis**: every platform that has genuinely opened itself to third-party code either (1) accepts
a single-tenant blast radius (WordPress) that does not apply to BizFirst, (2) declines to do it at
all and offers embed-only escape hatches instead (Webflow, and App Studio's own `content` widget
today), or (3) pays for a real sandboxing/review investment (Shopify) of the size Part 2(c) describes.
There is no precedent anywhere for "third-party widget types, lightly built, low cost" — that
combination does not exist in the wild for good reason.

---

## Part 4 — How the blog/help-docs/FAQ/etc. use cases actually map onto this

Confirmed against real, **live** plumbing (not just a design doc): `AIExt_AppPages` (Physical/Virtual
pages), `AppEngine.navigate({widgetID, entityType, dataID})`, and `RouteResolverService` are real,
applied-to-`data-ocean-platform-prod` code (`app-studio-pages-navigation.md`'s Phase 1 backend —
V011-V014 — is done, confirmed via `AppStudio\DevelopmentHistoryLog.md`'s 2026-08-26/27 entries, not
merely designed).

**Important nuance this research surfaced, worth correcting the premise slightly**: most of the 15
use cases' actual CRUD/data plumbing does not need a *new WidgetType* at all. The existing `form`
widget (`mode: 'list'` for the list view, `mode: 'view'`/`'edit'` for the detail view, bound to any
Atlas Forms `formId`) combined with a Virtual `AppPage` (`entityType` + a row-click-supplied
`dataID`) already mechanically implements "list widget + detail widget riding `entityType`/`dataID`
routing" **today, with zero new WidgetType, for any use case that is fundamentally a CRUD table with
a list/detail view** — a blog post table, an FAQ table, a testimonial table, a job posting table,
etc. are all, structurally, an Atlas Form + a Virtual Page. This *confirms* the task brief's
hypothesis about the right shape, and narrows what Phase 1's "widget kit" convention actually needs
to add value on top of: not the data/routing plumbing (already solved), but the **presentation
layer** — a blog list needs to look like a blog (card grid, excerpt, date, author, cover image), not
Atlas Forms' generic list-grid chrome; a FAQ needs an accordion, not a table. That presentation gap
is exactly what a config-driven widget kit (Part 2(b)) should target: a declarative "render this
`entityType`'s records as a blog-card-grid / FAQ-accordion / testimonial-wall" template, parameterized
by which fields map to which visual slots, riding the *same* underlying `form`/`AppPage` data layer
rather than replacing it. New, genuinely novel `WidgetType`s (Part 2(a)) become necessary only for
interaction shapes the `form` widget's list/edit/view modes can't approximate at all (e.g. a
pricing-table's tier-comparison layout, a podcast series' audio-player-per-episode list) — a
minority of the 15, not most of them.

---

## Part 5 — Phased recommendation

### Phase 1 (recommended starting point): (a) + (b), reusing the already-real `AIExt_WidgetTypes` catalogue

**What "bring your own widget type" should concretely mean for Phase 1** — for BizFirst's own team
first, extended to trusted partners once the convention is proven, explicitly **not** yet "anyone can
upload arbitrary code":

1. **Close the drift, don't add new architecture.** `AIExt_WidgetTypes` is already the real backend
   source of truth — wire `WidgetTypeCatalog.cs` (MCP layer) to read it instead of its own hardcoded
   dictionary, and add the missing `Configuration`-shape validation layer this research found absent
   (a small, per-type JSON-schema check, associated with each `AIExt_WidgetTypes` row — e.g. a
   `ConfigSchema` JSON column on that table, validated at `WidgetService.ValidateAsync` time). This
   alone turns "pluggable" from a half-true claim into an accurate one for the layers that already
   exist.
2. **A documented package/registration convention** (a runbook, mirroring `add-new-widget-type.md`'s
   own discipline) that says precisely: new `@app-studio/widget-{type}` package shape, required
   `*WidgetConfig.ts`/`*WidgetHandler.ts`/`*WidgetRenderer.tsx` exports, a `ConfigSchema` to register
   alongside the `AIExt_WidgetTypes` row, the `--app-var-*` theming contract as a hard requirement
   (not optional — per `content-widget-templates.md`'s own stated rule), and the three
   `.register()` call-sites that must be touched together (ideally consolidated into one shared
   bootstrap function during this phase, closing the registration-parity risk directly rather than
   documenting around it).
3. **The config-driven widget-kit layer (Part 2(b) / Part 4)** — a declarative template mechanism
   (JSON schema + a bounded set of rendering primitives: list-of-records card grid, detail-record
   view, accordion, carousel) that rides the *existing* `AppPage`/`entityType`/`dataID`/`form`-widget
   plumbing rather than requiring a new `WidgetType` per use case. This is where most of the blog/
   FAQ/testimonials/etc. ambition should land — build 2-3 of the 15 use cases this way first (blog,
   FAQ, testimonials are good proving cases — one needs card-grid+detail, one needs accordion, one
   needs a wall/carousel) to validate the primitive set before generalizing further.

This is a moderate, well-grounded extension of real, already-proven patterns — not a rebuild. It
also directly fixes a live product bug in the making: today, a widget type added to
`AIExt_WidgetTypes` alone is *invisible in the UI and unrenderable in the App Player*, which is a
confusing, worse-than-nothing half-state for whoever eventually uses that table for its intended
purpose. Phase 1 should close that gap even before any new widget type ships through it.

### Phase 2 (a separate, later initiative, NOT part of Phase 1): true external/third-party plugin support

Named honestly, per the task brief's own instruction not to soften this: this is closer to "design
and ship a second, sandboxed rendering runtime" than "extend the widget system." Before it can be
proposed as its own initiative, it needs, at minimum:

- A decided sandboxing model (iframe+postMessage vs. Web Component vs. Shopify-style host-owned
  injection slots — Part 3's synthesis suggests the Shopify model, where the host controls the actual
  injection points and the third party supplies data/behavior through a constrained bridge, is the
  only one of the three industry precedents that has actually been proven safe at scale for a
  multi-tenant SaaS platform).
- A theme-token bridge decision (how/whether `--app-var-*` reaches across whatever sandbox boundary
  is chosen — today it doesn't cross ANY existing boundary in this codebase, per §1.6's own documented
  bugs even for first-party editor surfaces missing the theme-scope attributes).
- A versioned, stable plugin API contract, a bundle integrity/signing scheme, and a real review/
  publish gate.
- A dedicated security review before any of it is exposed beyond BizFirst's own team, as its own
  gated milestone — not a checkbox at the end of Phase 1.

This should be scoped, estimated, and greenlit as its own initiative once Phase 1 has real usage data
(how many widget kits get built, which presentation gaps keep recurring) to inform what a plugin API
surface would actually need to expose.

---

## Sources

- `BizFirst.Ai.Mcp.Tools.AppStudio\Tools\WidgetTypeCatalog.cs`
- `BizFirst.Ai.AIExtension.Domain\Entities\WidgetTypeDefinition.cs`,
  `Interfaces\Repositories\IWidgetTypeRepository.cs`
- `BizFirst.Ai.AIExtension.Service\Services\WidgetService.cs` (`ValidateAsync`,
  `CreateAsync`/`UpdateAsync` overrides)
- `BizFirst.Ai.AIExtension.Service\DevelopmentHistoryLog.md` (2026-08-25/27/28 entries — the repeated
  frontend/backend allow-list drift history)
- `BizFirstFiDB\AppStudio\migrations\V017_UP_AIExt_WidgetTypes_Create.sql`,
  `BizFirstFiDB\AppStudio\DevelopmentHistoryLog.md`
- `BizFirstFiDB\BizFirstFiV3DB\...\dbo\Tables\AIExt_WidgetTypes.sql`
- `app-handlers-core\src\types\WidgetRecord.ts` (`WidgetType` union),
  `WidgetTypeRegistry.ts` (`WIDGET_TYPE_REGISTRY`)
- `widget-handlers-generic\src\WidgetRegistry.ts`
- `apps\app-player\src\App.tsx`, `apps\static-app-player\src\services\widgetRegistry.ts`,
  `app-studio-designer-components-react\src\preview\useLivePreviewEngine.ts` (the three
  `.register()` bootstrap sites)
- `app-studio-designer-components-react\src\modals\WidgetEditorFields.tsx` (the only real dynamic
  `import()` usage found, first-party/build-time only)
- `Documentation\Employees\agentic-development-engineers\app-development-engineer\rag\
  app-studio-rag\{site-building-lessons.md, theming.md, widgets\content-widget-templates.md,
  agent\add-new-widget-type.md}`
- `Documentation\Employees\agentic-development-engineers\app-development-engineer\design\targets\
  app-studio-pages-navigation.md` (Physical/Virtual `AppPage`, `entityType`/`dataID` routing —
  confirmed live via `AppStudio\DevelopmentHistoryLog.md`'s 2026-08-26/27 entries)
- Industry grounding: WordPress plugin model, Webflow Custom Code/embed model, Shopify Theme App
  Extensions/App Blocks/App Bridge, Notion embeds (general knowledge, cross-checked against Part 1's
  codebase findings rather than taken at face value).

## Constraints

- Research/design only — no code, schema, or data changed as part of this document.
- No commits/pushes without an explicit, same-message request.
