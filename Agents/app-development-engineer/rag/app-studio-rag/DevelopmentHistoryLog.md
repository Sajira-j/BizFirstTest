# App Studio RAG — Development History Log

## 2026-09-14 — Added `theming.md` (Task 89)

New Tier 1 doc: the full, current `--app-var-*` theme cascade reference (all 19 tokens from
`ThemeTokens.ts`, the two real integration shapes — `AppPlayer.tsx`'s 3-layer nested cascade vs.
WorkDesk's collapsed resolve-then-inject-once pattern — real before/after CSS from ChatDesk/
hil-ui-chat-window/WorkDesk/the Content Widget Templates, the two real theme-scope bugs found and
fixed 2026-09-11/12, and the `AppStudioTheme` preset catalog). Additive scope beyond the original
theming brief, per same-day follow-up requests: a full per-type catalog (real `Configuration` JSON
+ theme-participation + validation-gap notes) for all 18 real widget types, grounded in
`WidgetTypeCatalog.cs` and live `AIExt_Widgets` rows queried from `data-ocean-platform-prod`; and a
short section on the `StockAssets` local stock-image folder (163 files, `preview.html` browsing
aid) as a real-image source for widget authoring. `00-overview.md`'s Tier 1 index updated to point
to it. Existing `styling-and-common-properties.md`'s own theming summary (12-token version) is now
stale on the token count — left as-is rather than edited, since this new file is the authoritative,
current version and a future full pass could fold the two together.

## 2026-09-04 — Created (Task 51, superseding an earlier partial attempt)

Written from scratch, source-grounded directly against current `app-studio` package code (not the
DB `AIExt_WidgetTypes` description columns). Covers all 17 real `WidgetType` values with full
Tier 1 config docs — full first-pass coverage, unlike `workflow-nodes-rag`'s partial (18/107)
coverage of workflow nodes.

**Supersedes** `..\_archive\app-studio-rag-v1-2026-08-30\` (originally under
`agentic-coding\app-studio--automation-project\rag\v1\`), which covered only 10 of the (then-fewer)
widget types and predates the 7 media widgets (`image`/`video`/`audio`/`pdf` singles +
`image-gallery`/`video-gallery`/`audio-gallery`/`pdf-gallery`) added 2026-09-02, and the App Studio
Template wizard system (Task 22/23) added 2026-09-04. That set's two genuinely non-duplicated files
(`01-common-properties.md`, `02-style-properties.md` — AppSection shared fields and the full
Structured Style Builder system) were merged into this set as `styling-and-common-properties.md`
rather than left to rot in the archive.

Files: `00-overview.md` (Tier 0), `app-model.md`, `app-creation-flow.md`,
`styling-and-common-properties.md`, `widgets\{17 files}.md`, `agent\add-new-widget-type.md`.

Reorganized the same day (Task 51 follow-up, "agentic-development-engineers" restructure) from
`agentic-coding\app-studio-rag\` into its current home under
`agentic-development-engineers\app-development-engineer\rag\app-studio-rag\` — part of a
codebase-wide reorganization of Agent/MCP-Server/RAG artifacts by development-engineer role
(Workflow / Form / App). See `..\..\..\README.md` for the full reorganization rationale.
