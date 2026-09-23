# `content` — Content Widget

Static HTML, Markdown, or plain-text content. Source: `widget-handlers-content-widget\src\
ContentWidgetConfig.ts` / `ContentWidgetRenderer.tsx` — this is the actual ground truth; a doc
correction made earlier tonight (2026-09-10) against the MCP validator instead of the renderer was
itself wrong and is reverted here. See "History of this doc's own bug" below — read it before
trusting any single source in this codebase blindly.

## Config

The real shape, per `ContentWidgetConfig` (`interface ContentWidgetConfig { content: string;
format: 'html' | 'markdown' | 'text'; ... }`): **two separate fields**, not a flattened
per-format key.

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `content` | string | Yes | — | The actual text/markup, regardless of format. |
| `format` | `'html'` \| `'markdown'` \| `'text'` | Yes | — | Which of the three `content` is. |
| `allowScripts` | boolean | No | `false` | Whether embedded scripts in HTML content execute. |
| `isDynamicContent` | boolean | No | unset | **Not yet consumed by the renderer.** Reserved for a future Expression Studio integration — when true, `content` is meant to be evaluated as an expression rather than literal text. Do not generate a workflow that relies on this actually working today. |
| `contentCacheEnabled` / `contentCacheExpiry` | boolean / number (seconds) | No | unset | Also reserved for the same future dynamic-content work; only meaningful once `isDynamicContent` is real. |

## Example

```json
{ "content": "# Welcome\n\nThis page is under construction.", "format": "markdown" }
```

## Gotchas

- The dynamic-content fields (`isDynamicContent`/`contentCacheEnabled`/`contentCacheExpiry`) exist
  in the type shape but have NO real behavior yet — don't generate content assuming expression
  evaluation works. Treat `content` as always-literal for now.
- The resolved value of a future dynamic-content evaluation is explicitly NOT stored on this
  config (a deliberate decision) — don't add a `cachedContent`-style field yourself.
- `ContentWidgetRenderer` renders `format: 'text'` as plain pre-wrapped text and anything else as
  sanitized HTML (Markdown is converted to HTML first) — an unrecognized/missing `format` value
  falls through to the HTML branch with empty `sanitizedHtml`, which is exactly the failure mode
  below: the widget mounts (you'll see its wrapper `<style>` tag in the DOM) but shows nothing.

## History of this doc's own bug — read this before trusting one source over another

This doc briefly (same night, a few hours) claimed the OPPOSITE of what's above — a flattened
`{markdown: "..."}`/`{html: "..."}`/`{text: "..."}` shape — because that flattened shape was
"confirmed" against a real, live MCP validator call that accepted it. **The validator was the one
that was wrong**, not this doc's original claim: `WidgetTypeCatalog.Validate()`'s "content" case
checked for `configuration["markdown"|"html"|"text"]` directly, and `create_widget`'s own
`[Description]` docstring gave the same wrong example — both now fixed (2026-09-10) to check/show
`content`+`format`. A config shape "passing validation" is not the same as "the real renderer will
show it to a real user" — this was live-reproduced on a real app (AppID 1064): a `create_widget`
call with `{"markdown": "..."}` succeeded, the real `AppWidget`/`Widget` rows were created
correctly, the widget mounted in the real Player's DOM (confirmed via `data-widget` attribute
inspection) — and rendered zero visible text, because `ContentWidgetRenderer` was reading
`config.content`/`config.format`, which were never set. **The lesson, not just the fact**: when a
tool's own validator and its own renderer disagree about a config shape, the renderer is the real
authority (it's what an end user actually sees) — a validator can be just as wrong as a doc, and a
successful `create_widget` response proves the row was written, never that it will actually render.
