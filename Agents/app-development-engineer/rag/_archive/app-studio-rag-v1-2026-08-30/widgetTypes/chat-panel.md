# `chat-panel`

Source: `WidgetRecord.ts`'s own doc comment, scanned 2026-08-30. **Real finding, not just
unverified**: this session's directory scan of `packages/widget-handlers-*-widget` found NO
`chat-panel`-named package (the 9 real directories are: `content`, `core`, `form`, `generic`,
`hil-inbox`, `notifications`, `page-navigation`, `signin`, `site-branding`, `workflow-template`).
`chat-panel` is declared in the `WidgetType` union but may not have a dedicated handler package as of
this scan — it may be implemented inside another package, registered dynamically, or simply not yet
wired into the runtime registry. **Do not assume this widget type is actually usable/registered
without re-confirming `registry.get('chat-panel')` resolves to something real** (grep
`useLivePreviewEngine.ts`/`app-player`'s own registration calls for a `registry.register('chat-panel',
...)` line) before generating content that relies on it.

Mounts a real chat window via the extracted `@bizfirst/chatdesk-chat-window` package (reused, not
reimplemented) against a resolved App/Process target. This is the one real widget type this scan
found with the LEAST independent verification — no `WidgetRenderResult` variant for it was read
directly this pass (the discriminated union file was only read partially). Before generating config
for this widget type, read `packages/widget-handlers-*` for its real handler package and
`WidgetRenderResult`'s full union in `widget-handlers-core/src/IWidgetHandler.ts` for its actual
render-result shape — do not guess a shape from this entry alone.
