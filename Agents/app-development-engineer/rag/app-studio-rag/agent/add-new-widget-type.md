# Add a New Widget Type to `app-studio-rag`

Use this runbook whenever a widget type needs a Tier 1 doc — a new `WidgetType` was added to the
codebase, or an existing one's config changed. Written to be handed directly to a fresh agent with
zero prior context. Mirrors the discipline `..\..\..\workflow-development-engineer\rag\
workflow-nodes-rag\agent\add-new-node-type.md` established for its own doc set — same two-tier
shape, same "ground truth is the code" rule, adapted for App Studio widgets.

## The one non-negotiable rule

**Read the real `*WidgetConfig.ts` interface first.** Read the widget's `*WidgetHandler.ts`/
`*WidgetRenderer.tsx` for behavior. Read `AIExt_WidgetTypes`'s DB description columns only
afterward, if at all, as a factual comparison — never as a template for what the doc should say.
The DB description is display metadata for a UI label, not a config contract.

## Step 1 — find the real source

1. Locate the widget package: `BizFirstAiStudio\src\app-studio\packages\widget-handlers-{type}-
   widget\src\`.
2. Read `*WidgetConfig.ts` — the real TypeScript interface. Every field, its type, whether it's
   required (no `?`), and its real default (from the renderer's fallback logic, not guessed).
3. Read `*WidgetHandler.ts` for `render()`'s behavior and whether `resolveConfig()`/`load()` does
   anything non-trivial.
4. Check `app-handlers-core\src\types\WidgetTypeRegistry.ts` for the real label/description already
   used by the creation UI, and `WidgetRecord.ts`'s `WidgetType` union doc comment for any dated
   context on why/when the type was added.
5. If the widget shares config with siblings (e.g. the four gallery widgets share
   `GalleryWidgetConfig`), find and read the shared base type too — document the shared shape once,
   in the base type's own doc, and have sibling docs reference it (see `widgets\image-gallery.md` /
   `video-gallery.md`/`audio-gallery.md`/`pdf-gallery.md` for the worked example).

## Step 2 — write the Tier 1 doc

Default to one flat `widgets\{type}.md` file — App Studio widget configs are consistently flat
property lists (no config type here has qualified for the split-doc pattern
`workflow-nodes-rag` uses for `ai-agent`/`flow-ai-agent` yet; if a future widget type's config grows
several genuinely independent sub-objects, apply that same split pattern and note it in
`00-overview.md`'s index).

```
# `{widget-type-code}` — {Label}

One paragraph: what it does, whether it's safe-default (drag-to-place) or modal-required
(click-to-configure — check `addWidgetActions.ts`'s `SAFE_DEFAULT_WIDGET_TYPES`), what package it's in.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
... every real field ...

## Example

```json
{ ...a realistic Configuration for typical use... }
```

## Gotchas
- anything a config-authoring agent would get wrong by guessing instead of reading this doc
- known limitations (e.g. no server-side thumbnails, a field that exists but isn't consumed yet)
```

## Step 3 — update the index

In `00-overview.md`: add one row to the widget type table (code / label / what it does / config?
yes-no / doc path). Do not add per-field detail there — Tier 0 stays lean, same rule the workflow
node RAG set enforces for itself.

## Step 4 — independent review

Don't treat your own doc as verified just because you read the source carefully. A second pass
(fresh agent, or a careful self-review after a break) re-reading the real source independently and
checking the field list/types/gotchas against it is the same discipline `refreshFromCodeToDoc@agent.md`
requires for Atlas Forms and `add-new-node-type.md` requires for workflow nodes.

## Constraints

- Documentation only — do not modify any widget-handler code as part of writing a Tier 1 doc.
- Do not commit or push anything without being explicitly asked in that specific request.
- Ground every claim in real, currently-read source — not in a prior doc, not in
  `AIExt_WidgetTypes`, not in this runbook's own examples.
