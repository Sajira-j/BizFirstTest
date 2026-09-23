# App Studio — Full MCP Tool + Widget Type Test Plan

Mirrors the Atlas Forms testing approach (`form-development-engineer\test-results\test2-control-reference\00-test-plan.md`): a real test plan reviewed before execution, then executed via the real MCP API-key path and verified visually in the real Player via the Chrome extension — not just trusted from tool responses.

## Part 1 — MCP tool coverage (mostly done, one gap closing now)

All 16 App Studio MCP tools have been exercised with real calls tonight (`test-results\app-studio-mcp-coverage\app-studio-mcp-coverage.html`): `list_apps`, `get_app`, `create_app`, `create_project_with_app`, `create_app_from_template`, `update_app`, `delete_app`, `list_pages`, `create_page`, `update_page`, `delete_page`, `list_widgets_in_app`, `create_widget`, `update_widget_placement`, `list_widget_types`, `create_section` (built tonight to close a real gap). `create_app_from_template`'s real success path (vs. just its error path) is being closed by a separate in-flight agent as of this writing — check that report before assuming Part 1 is 100% done.

## Part 2 — All 18 widget types (not yet done, this is the new work)

Same batching principle proven for Atlas Forms controls: multiple widgets per app/page, not one app per widget type. Two real, already-fixed building blocks make this straightforward now: `create_section` (widgets need a real section to attach to) and the corrected `content` widget shape (`{content, format}` — the bug found and fixed earlier tonight).

| # | Widget type | Config needed | Real dependency | Batch with |
|---|---|---|---|---|
| 1 | `content` | `{content, format}` | none | — (already proven working) |
| 2 | `label`/text-ish equivalents n/a (App Studio has no separate label widget — content covers it) | | | |
| 3 | `image` | `src`, `alt`, sizing | a real (license-safe) image URL — reuse the MJ site's own Wikimedia URL or another verified-free one | page 1 |
| 4 | `video` | `src` | a real, freely-embeddable video URL (e.g. a Creative-Commons-licensed or self-hosted test clip) | page 1 |
| 5 | `audio` | `src` | same licensing care as video | page 1 |
| 6 | `pdf` | `src` | any real, linkable PDF URL | page 1 |
| 7 | `page-navigation` | orientation, nesting | needs real pages to link to — use the MJ site's own pages | page 2 |
| 8 | `hil-inbox` | none (auth/tenant only) | none | page 2 |
| 9 | `signin` | optional | none | page 2 |
| 10 | `notifications` | optional | none | page 2 |
| 11 | `site-branding` | optional | none | page 2 |
| 12 | `form` | `formId` (required) | a real existing FormID — reuse one of tonight's Atlas Forms test forms (10001133-10001140) | page 3 |
| 13 | `workflow-template` | `executionTemplateID` (required) | a real `Process_ExecutionTemplates` row — **check whether one exists; may not, since tonight's work created a raw workflow, not an execution template.** If none exists, document as a real blocker rather than fabricating one. | page 3 |
| 14 | `workflow-template-category` | `executionTemplateCategoryID` (required) | same dependency risk as #13 | page 3 |
| 15 | `chat-panel` | `processID` (required) | a real `Process` row — tonight's sample workflow (Process 1073) can supply this | page 3 |
| 16-18 | `image-gallery`, `video-gallery`, `audio-gallery`, `pdf-gallery` (4, not 3 — overview lists all four) | none directly, but **query LIVE public assets flagged `IsPublicAsset=true`** | **No MCP tool exists to upload/flag an asset as public.** This is a real, likely blocker — check whether any real `IsPublicAsset=true` rows already exist in this tenant's asset table before assuming these are testable at all via MCP alone. | page 4 (or document as blocked) |

**Known risks, read before executing:**
- Widgets #13/#14 (execution-template widgets) may have no real target to reference yet — check first, don't fabricate IDs (a fake ID would create a widget that looks fine via MCP but is broken in the real UI, exactly the class of bug this whole testing effort exists to catch).
- Widgets #16-19 (the four gallery types) depend on real public-asset data this session has no tool to create — check for existing rows first; if none exist, document plainly as "not testable via MCP today, needs an asset-upload tool that doesn't exist yet" rather than skipping silently.
- Reuse the **existing** cross-tenant reference validation gap already documented in `app-model.md` (the 4 reference-holding widget types only get presence checks, not existence/tenant-ownership checks) — this test pass is a good opportunity to confirm that gap is still real and re-flag it, not just re-discover it.

## Execution plan

One form/page per row-group above (4 pages total: media, chrome/nav, cross-domain-reference, galleries), built on a fresh test app (not the MJ site — keep that as the polished deliverable, use a dedicated `AppStudioWidgetTypeTest` app for this so nothing here risks the MJ site's own state). Screenshot every widget live in the real Player via the Chrome extension (permission already granted), verify each widget's real DB row via `sqlcmd`, and produce one `test-results\app-studio-widget-test\{widgetType}.html`-style page per type, same labeling standard as the Atlas Forms control reference pages (every screenshot captioned with exactly which config is shown).

## Sequencing

Execute after the currently in-flight Servers MCP and `create_app_from_template` gap-closing agents finish (one agent at a time, per tonight's budget correction) — this plan is ready to hand to the next dispatch.
