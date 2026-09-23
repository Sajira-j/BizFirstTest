# `content`

Source: `packages/widget-handlers-content-widget/src/ContentWidgetConfig.ts` +
`ContentWidgetHandler.ts`, scanned 2026-08-30. Renders free-form rich text/HTML/Markdown, sanitized
before render. The most commonly used widget type for page body content (qoboto's entire site is
built almost entirely from `content` widgets).

## Config

| Property | Type | Default | Notes |
|---|---|---|---|
| `content` | string | required | raw HTML or Markdown source, per `format` |
| `format` | `ContentFormat` (`'html'\|'markdown'\|'text'`) | `'html'` | Markdown is converted to HTML via `marked` (`breaks:false, gfm:true` — CommonMark-style: blank line between paragraphs, not single-newline soft breaks) BEFORE sanitization; `'markdown'` never reaches the render result as-is — it's normalized to `'html'` in the returned `WidgetRenderResult.format` |
| `allowScripts` | boolean? | `false` | when false (the default, and the only value used in practice), `<script>` tags are stripped by the sanitizer |

Dynamic-content fields (`expression`-related) exist on the type as of 2026-08-25 but are **not yet
consumed by the renderer** — config-shape-only, reserved for a later Expression Studio integration.
Do not rely on them having any render-time effect today.

## Render result shape

```ts
{ type: 'content', format: 'html'|'markdown'|'text', sanitizedHtml: string }
```

`sanitizedHtml` is real, DOMPurify-sanitized output — genuine `<h1>`/`<h2>`/`<img alt="...">` etc. can
exist in it. This is what the SEO checklist parses for heading structure/alt-text coverage (see
`architecture.md`'s digital-assets-library/SEO work).

## Editing UX (as of this session's Designer work)

The click-to-edit Designer overlay (see `design-and-plan.md` Task 2) mounts a Tiptap rich-text editor
directly as this widget type's inline "landing tab" for `format==='html'` — save-on-dismiss, no
explicit Save button. For `'markdown'`/`'text'`, the landing tab is a plain textarea (no WYSIWYG
claim, since those formats aren't WYSIWYG by nature).

## Examples

```json
{ "content": "<h1>Welcome</h1><p>Real content here.</p>", "format": "html", "allowScripts": false }
```

```json
{ "content": "# Welcome\n\nReal content here.", "format": "markdown" }
```
