# App Studio RAG — Overview (Tier 0 — always loaded)

Source of truth: real App Studio source code under
`BizFirstAiStudio\src\app-studio\packages\` — each widget's own `*WidgetConfig.ts` (the real
TypeScript config shape) and `*WidgetHandler.ts`/`*WidgetRenderer.tsx` (real behavior), plus
`app-handlers-core\src\types\` for the core data model (`WidgetRecord.ts`, `WidgetTypeRegistry.ts`,
`AppPageRecord.ts`) and `app-studio-api-client-js\src\` for real API endpoints. **Not**
`AIExt_WidgetTypes`'s DB description columns or any registry UI label alone — those are display
metadata, not the authoritative config shape. Scanned/written 2026-09-04. Mirrors the two-tier
retrieval pattern and sourcing discipline established by `..\bizfirst-ai-mcp-servers-spec\
workflow-nodes-rag\` — read that set's own `00-overview.md` for the fuller rationale if unfamiliar.

**Purpose**: this doc set exists to let an AI agent create a working App Studio app — pick/place
the right widgets with correct config, understand the App/Page/Widget data model, and use the real
creation APIs (including the new template-based wizard) — grounded in real, current code, not
guessed field names.

**This file is Tier 0 — the only part of this doc set that should ever be always-loaded into an
agent's context.** It stays a lean index: what widget types exist, one-line descriptions, and
pointers to Tier 1 detail. It must never accumulate per-widget config field detail — that lives in
`widgets\{type}.md`, retrieved only when a request actually implies that widget type is relevant.

## Two-tier retrieval — how an agent should actually use this

- **Tier 0 (this file, always loaded):** widget type index below, one-line descriptions, and the
  three core-concept pointers (`app-model.md`, `app-creation-flow.md`).
- **Tier 1 (retrieved on demand, one file per concern):**
  - `app-model.md` — the App/Page/AppWidget/Widget data model and real CRUD API endpoints. Load
    this whenever the request involves creating/modifying app structure, not just placing one
    widget.
  - `app-creation-flow.md` — how a new App gets created (Project-unified flow, the wizard, template
    export/import). Load this when the request is "create a new app."
  - `widgets\{type}.md` — full config field table + real example + gotchas for exactly the widget
    type(s) the current request implies. There are 17 real widget types; fetch only the ones
    actually needed for the app being built, not all 17 preemptively.
  - `agent\add-new-widget-type.md` — the runbook for extending this doc set when a new widget type
    is added to the codebase.

## Widget type index

All 17 real `WidgetType` values (`app-handlers-core\src\types\WidgetRecord.ts`), one line each.
Every type below has a Tier 1 doc — this doc set has full first-pass coverage, unlike
`workflow-nodes-rag`'s partial coverage.

| Code | Label | What it does | Config? | Tier 1 doc |
|---|---|---|---|---|
| `form` | Form Widget | Create/edit/view/list records from an Atlas Forms form — the generic-CRUD building block. | Yes (`formId` required) | `widgets\form.md` |
| `content` | Content Widget | Static HTML, Markdown, or plain-text content. | Yes | `widgets\content.md` |
| `workflow-template` | Workflow Agent | One AI agent/execution template, with Execute or Chat Now. | Yes (`executionTemplateID` required) | `widgets\workflow-template.md` |
| `workflow-template-category` | Workflow Category | A grid of every agent in one execution-template category. | Yes (`executionTemplateCategoryID` required) | `widgets\workflow-template-category.md` |
| `chat-panel` | Chat Panel | An embedded chat window for triggering/conversing with one process. | Yes (`processID` required) | `widgets\chat-panel.md` |
| `page-navigation` | Page Navigation | A placeable menu of app pages, vertical or horizontal, nested-page aware. | Yes | `widgets\page-navigation.md` |
| `hil-inbox` | HIL Inbox | A human-in-the-loop actionable inbox — approvals, forms, tasks. | None (auth/tenant only) | `widgets\hil-inbox.md` |
| `signin` | Sign In / Sign Out | Sign-in link when signed out; user menu with sign-out when signed in. | Yes (optional) | `widgets\signin.md` |
| `notifications` | Notifications | A notifications bell with unread count and dropdown list. | Yes (optional) | `widgets\notifications.md` |
| `site-branding` | Site Branding | The App's own logo and name, side by side. | Yes (optional) | `widgets\site-branding.md` |
| `image` | Image | A single photo — URL, alt text, caption, object-fit, optional link. | Yes (`imageUrl` required) | `widgets\image.md` |
| `video` | Video | A single video player — URL, poster/caption, autoplay/loop/controls. | Yes (`videoUrl` required) | `widgets\video.md` |
| `audio` | Audio | A single audio player — URL, title/caption, autoplay/loop. | Yes (`audioUrl` required) | `widgets\audio.md` |
| `pdf` | PDF | A single PDF — URL, optional title, link or inline embed. | Yes (`pdfUrl` required) | `widgets\pdf.md` |
| `image-gallery` | Image Gallery | Live, filterable collection of public images — 5 themes. | Yes | `widgets\image-gallery.md` |
| `video-gallery` | Video Gallery | Live, filterable collection of public videos — 3 themes. | Yes | `widgets\video-gallery.md` |
| `audio-gallery` | Audio Gallery | Live, filterable collection of public audio tracks — 2 themes. | Yes | `widgets\audio-gallery.md` |
| `pdf-gallery` | PDF Gallery | Live, filterable collection of public PDFs — 2 themes, link or inline embed. | Yes | `widgets\pdf-gallery.md` |

The four single-media widgets (`image`/`video`/`audio`/`pdf`) each hold one FIXED URL; the four
gallery widgets (`image-gallery`/`video-gallery`/`audio-gallery`/`pdf-gallery`) instead run a LIVE
QUERY against public assets at render time (never a fixed list) — don't conflate "gallery" with
"a list of specific chosen items," it isn't one.

## Core concepts — load these before generating app structure

- **`app-model.md`**: the real App → AppPage → AppSection → AppWidget → Widget hierarchy, what
  fields each level owns, and the real CRUD endpoints (`/apps`, `/apps/{id}/pages`,
  `/apps/{id}/widgets`, `/widgets`).
- **`app-creation-flow.md`**: how a new app actually gets created today — the unified Project→App
  flow, the wizard (Empty vs. Template), and the template export/import mechanism (clone with new
  IDs, not references).
- **`styling-and-common-properties.md`**: AppSection's shared fields, the `IWidgetHandler` contract,
  and the full structured Style Builder system (StyleSlot levels, the ~90-property `StyleProperties`
  shape, the `css` escape hatch's hard sanitizer rule). Load this whenever generating any styling.
- **`theming.md`**: the full, current `--app-var-*` theme cascade contract (all 19 tokens,
  DOM-scoping rules, how to integrate a new consumer, real before/after CSS, common mistakes), a
  per-widget-type catalog with real `Configuration` examples and theme-participation/validation-gap
  notes for all 18 widget types, the `AppStudioTheme` preset system (`DataTemplateTypeID=54`), and
  the `StockAssets` local stock-image library. Load this whenever theming a `content` widget,
  picking/configuring any widget type, or sourcing a real image for a widget.
- **`site-building-lessons.md`**: composition/quality lessons for building a full MULTI-PAGE,
  MULTI-WIDGET app via the MCP tools — the `allowScripts` bug that silently strips a content
  widget's entire `<style>` block (the #1 cause of a "looks unstyled/thin" build), which layout
  widgets (`site-branding`/`page-navigation`) do and don't participate in `--app-var-*` theming,
  section/sequencing gotchas (`create_section` before `create_widget`), spacing/typography/imagery
  rhythm lessons, and why a live theme-switch test (not just a structural config check) is required
  before calling a themed build done. Load this whenever the request is "build a site/app" (not just
  one widget), or when a built site is reported as looking unstyled/thin/unformatted.

## Security/scope rules that apply across every widget

- Gallery widgets and the Media Source picker (used by all four single-media widgets' URL fields)
  only ever query/show assets flagged `IsPublicAsset === true` — this is a hard, non-negotiable
  boundary in the real backend query, not a UI-only filter. Never generate a workflow that assumes
  a private asset is reachable through these paths.
- No server-side image/video/audio/PDF thumbnail generation exists anywhere in this pipeline —
  every media widget serves the original uploaded file. Don't assume a `thumbnailUrl`-style field
  exists on any media config; there isn't one.
