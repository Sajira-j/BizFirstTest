# App Studio Designer — Undo/Redo Design

Status: Design only. No code changes in this pass.
Scope: `app-studio-designer` app, `@app-studio/store-react`'s `useAppSelectionStore`
(`packages/app-studio-store-react/src/stores/app-selection.store.ts`).

## 1. What already exists (don't rebuild this)

`app-selection.store.ts` + `VersionsApiClient` (`packages/app-studio-api-client-js/src/versions.client.ts`)
already provide a **durable, server-persisted, coarse checkpoint** system:

- `saveVersion(label?)` → `POST /apps/{id}/versions` — snapshots the whole app.
- `VersionPanel.tsx` (`packages/app-studio-designer-components-react/src/versions/`) lists and
  restores (`POST /apps/{id}/versions/{versionId}/restore`) these snapshots.
- This is a **"Save As a named checkpoint / roll back to it"** feature — the same category as
  Google Docs version history or a git tag. It is explicitly out of scope here and must not be
  touched or conflated with undo/redo. Undo/redo below is a *session-local, in-memory, Ctrl+Z*
  feature; Versions remains the durable long-term mechanism.

## 2. The real blocker: two categories of store action, discovered by reading the code

Every mutating action in `app-selection.store.ts` falls into one of two categories, and this
split — not UI design — is what determines what undo/redo can safely cover today.

**Category A — pure client-side, deferred to the main Save button.**
Mutates `layout` only, sets `isDirty: true`, issues **zero** network calls. The only action that
is 100% Category A today is:
- `moveSection` (swaps siblings in `layout.appSections` via `swapSectionInTree`, synchronous, no
  `await` anywhere in the function body).
- Section-style edits made through `SectionDetailsContent`'s `StylePropertiesTab` in
  `CanvasPanel.tsx` (the `patchSection` path that sets `sectionContainer`/`sectionBackground`) —
  also pure `layout` mutation, no immediate API call.

**Category B — fires a real backend call as part of the same user gesture.**
Everything else, including two actions whose NAME suggests they're layout-only but whose
implementation is not:
- `removeAppWidget`, `moveAppWidget`, `reorderWidgetsInSection` — call
  `AppWidgetsApiClient.delete`/`.update` immediately (DisplayOrder/sectionName are relational
  columns on `AppWidget`, not part of the `layout` JSON blob).
- `removeSection` — deletes the section from `layout` (Category A) **and** cascades to
  `AppWidgetsApiClient.delete` for every widget in the section and its descendants (Category B),
  in the same store action.
- `renameSection` — same split: updates `layout` locally, but first calls
  `AppWidgetsApiClient.update({ sectionName })` on every affected widget, with its own
  allSettled/rollback-on-partial-failure logic (see the function's own comments).
- Widget config edits via `WidgetEditorFields.tsx`'s `handleSave` — calls
  `AppWidgetsApiClient.update(selectedAppId, appWidgetID, {...})` directly, independent of the
  main Save button.

**Implication:** a naive "wrap every store call in an undo stack" design is unsafe. Undoing a
Category B action after the fact means either (a) re-issuing an inverse network call — which can
itself fail, and for a delete means *re-creating* the widget under a **new** `AppWidgetID`,
silently breaking anything that still references the old one (the just-closed Widget Details
panel, an in-flight drag, another browser tab) — or (b) not offering undo for it at all. Given
this codebase's own established pattern for partial failure (allSettled + best-effort rollback +
a thrown error the caller must surface, used identically in `moveAppWidget`/`renameSection`/
`removeSection`/`reorderWidgetsInSection`), silently claiming "undone" when a compensating call
fails would be a regression from that standard, not an enhancement.

## 3. Decision: client-side, in-memory, immer-patch stack — Save is still the durable checkpoint

- **In-memory only, not persisted per-keystroke.** Matches Wix's own model (undo is a live-session
  affordance; Save/Publish/Version snapshot are the durable checkpoints) and avoids a write-heavy
  "persist every undo step" design nobody asked for. The stack is lost on page reload — acceptable,
  since `isDirty` already means "reload will discard this" today; undo/redo doesn't change that
  contract, it just makes the current session's edits more forgiving.
- **Immer patches, not full-state snapshots.** `immer` is already a workspace catalog dependency
  (`BizFirstAiStudio/pnpm-workspace.yaml`, `immer: ^10.0.0`) — no new dependency. Use
  `produceWithPatches(layout, recipe)` at each Category-A mutation site to capture
  `{ patches, inversePatches }` instead of `structuredClone`-ing the whole `layout` object per
  step. A real qoboto-sized `layout` JSON is ~15-18KB (confirmed by inspecting `AIExt_Apps.
  Configuration` for AppID 1526 directly) — cheap to clone outright at this scale, so the patch
  choice here isn't primarily about memory, it's about **getting a native inverse operation for
  free** (`inversePatches`) instead of hand-writing an undo function per action type, and about
  the stack scaling correctly once undo coverage later grows to include larger/nested state.
- **Stack shape:**
  ```ts
  interface UndoEntry {
    patches: Patch[];          // forward (redo) — layout only, v1 scope
    inversePatches: Patch[];   // backward (undo)
    selectionBefore: { sectionKey: string | null; widgetId: number | null };
    selectionAfter: { sectionKey: string | null; widgetId: number | null };
    label: string;             // e.g. "Move section", "Edit section style" — for a future visible history list
  }
  // undoStack: UndoEntry[], redoStack: UndoEntry[] — new store slice, not persisted, cleared on selectApp()/loadSelectedApp()
  ```
  `undo()`: `applyPatches(layout, top.inversePatches)`, restore `selectionBefore`, push entry onto
  `redoStack`. `redo()`: the mirror. A new Category-A mutation clears `redoStack` (standard
  undo/redo semantics — a fresh edit invalidates the redo branch).

- **Interaction with `saveVersion`/Save:** independent, not reset by either. Saving does not clear
  the undo stack (a user should be able to Save, keep editing, and still Ctrl+Z the pre-save edit —
  this matches every real editor, Wix included). The stack IS cleared on `selectApp()`/
  `loadSelectedApp()` (switching apps, or a hard reload) since it has no meaning across a different
  `layout` object.

- **Category B stays out of the undo stack in v1 — explicitly, not silently.** Buttons/menu items
  for actions that aren't undoable (delete widget, reorder widgets, rename section, widget config
  save) should NOT appear to support Ctrl+Z; there is no partial/best-effort "sort of works" middle
  ground here given the ID-stability and partial-failure risks in §2. This is a real, disclosed
  product limitation for v1, not an implementation shortcut to quietly work around later.

## 4. Path to covering Category B (Phase 2, separate effort — not this pass)

The only way to make `removeAppWidget`/`moveAppWidget`/`reorderWidgetsInSection`/
`renameSection`/widget-config-save genuinely undoable is to **stop firing their API calls
immediately** and instead fold them into the same staged/dirty `layout`+`appWidgets` local-state
model that Category A already uses, deferring all persistence to `saveLayout()` (which would need
to become the one place that reconciles `appWidgets` additions/removals/reorders against the
backend, not just the `layout` JSON blob). This is a materially larger, separate architectural
change — it touches five store actions' existing, already-hardened allSettled/rollback logic and
`WidgetEditorFields.tsx`'s save path, all of which were deliberately built this session to persist
immediately. Recommend scoping this as its own follow-up design/implementation cycle once v1 undo
ships and its actual usage/pain points are known, rather than bundling a store-wide persistence
model change into "add undo/redo."

## 5. UI spec (v1)

- **Keyboard:** `Ctrl+Z` / `Cmd+Z` = undo, `Ctrl+Shift+Z` / `Cmd+Shift+Z` (and `Ctrl+Y` as a Windows-
  convention alias) = redo. Scoped to the Designer canvas only (not global `window` listener) —
  attach on the same container `CanvasPanel.tsx` already owns, so typing in an unrelated text input
  elsewhere in the app never triggers it; standard `contentEditable`/`<input>`/`<textarea>` focus
  check before handling, same guard pattern any rich-text-adjacent shortcut needs.
- **Toolbar:** two small icon buttons in `DesignerToolbar.tsx`'s left/center cluster (undo/redo
  arrows, disabled when the respective stack is empty), positioned near `SaveStatus` since both are
  "state of my edits" indicators. Tooltip shows the entry's `label` ("Undo: Move section").
- **Visual scrubbable history timeline** (Wix has one): out of scope for v1. The `label` field is
  captured now specifically so a future timeline/list view (Phase 2, once Category B is covered and
  the stack is large enough to be worth visualizing) doesn't require a data-model change later —
  but building that list UI now, for a stack that only covers two action types, isn't worth it yet.

## 6. Phased implementation plan

| Phase | Scope | Effort estimate |
|---|---|---|
| **1** | New `undoStack`/`redoStack` slice + `undo()`/`redo()` actions in `app-selection.store.ts`; convert `moveSection` and the section-style `patchSection` path to `produceWithPatches`; keyboard handler + toolbar buttons; clear-on-`selectApp` wiring. | 2-3 days |
| **2** (separate effort, own design pass) | Rework `removeAppWidget`/`moveAppWidget`/`reorderWidgetsInSection`/`renameSection`/widget-config-save to stage locally and defer persistence to `saveLayout()`; extend the patch stack to cover `appWidgets`; re-validate every existing partial-failure/rollback code path under the new deferred model. | 1.5-2 weeks — genuinely the larger, riskier piece; needs its own design doc before starting |
| **3** (optional, post-Phase 2) | Visible scrubbable history list using the already-captured `label` field. | 2-3 days |

Phase 1 ships real, safe value (the two actions users will reach for undo on most while actively
laying out a page — reordering and restyling sections) without touching any of the carefully-built
immediate-persistence/rollback logic elsewhere in the store.

## Build Progress — 2026-08-30

**Phase 1: shipped.** All items in the Phase 1 scope from §6 are implemented:

- `app-selection.store.ts`: `undoStack`/`redoStack` state, `UndoEntry`/`UndoSelection` types,
  `mutateLayoutWithUndo(recipe, label)` (immer `produceWithPatches`), `undo()`/`redo()`
  (`applyPatches`), cleared on `selectApp`/`loadSelectedApp`. `moveSection` rewritten to go through
  `mutateLayoutWithUndo`.
- `CanvasPanel.tsx`'s (now `details/SectionDetailsContent.tsx`'s, post Task-2-extraction)
  `patchSection` rewritten the same way — the second Phase 1 mutation path.
- `DesignerToolbar.tsx`: undo/redo `IconButton`s next to `SaveStatus`, disabled-when-empty, tooltip
  shows the top entry's `label`.
- `AppShell.tsx`: Ctrl+Z/Ctrl+Shift+Z/Ctrl+Y keyboard handler, gated on an app being selected and a
  focus guard — placed here rather than `CanvasPanel.tsx` (this doc's own §5 suggestion) because
  that component is being retired by the concurrently-in-progress Task 2 redesign; `AppShell` is
  the stable root across both the current and Task 2's simplified body-switch.
- `immer` added as a real dependency: this nested `app-studio` pnpm workspace didn't have it in its
  own catalog (only the top-level `BizFirstAiStudio` one did) — added an entry there first, matching
  the existing convention for this exact class of gap (see that catalog file's own "marked" comment).

**Verification**: `pnpm typecheck` is clean for `@app-studio/store-react`. It currently fails for
`@app-studio/designer-components-react` and `app-studio-designer`, but exclusively on pre-existing
errors inside `CanvasPanel.tsx` traceable to the concurrently-running Task 2 fork's own in-progress
edit (duplicate `SectionDetailsContent`/`WidgetDetailsContent` declarations, dangling references to
since-relocated imports) — confirmed none of the reported errors touch this feature's files.
Interactive live-verification (move a section, Ctrl+Z, Ctrl+Shift+Z) was attempted but inconclusive
for reasons outside this work's own control: the shared Designer browser tab was mid-frozen from
Task 2's in-flight broken bundle on the first attempt, and on retry showed signs of concurrent
interactive use by another agent. Confirmed instead via screenshot: the toolbar buttons render
correctly, and Layout Design mode correctly loads the real qoboto layout (11 sections) despite
Task 2's concurrent typecheck errors. **Recommend a follow-up live click-test** (move a section,
undo, redo, confirm the canvas/left list/live preview all agree) once Task 2 lands and the shared
browser session is free — not yet done.

**Not done (by design, per this doc's own Phase 2 scoping)**: Category B actions remain uncovered.
No git commits made — all changes are uncommitted, awaiting review.
