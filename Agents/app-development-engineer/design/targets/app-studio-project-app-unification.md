# Target: App Studio — Project/App unification (reuse Flow Studio's Project components)

## Problem statement

App Studio (`BizFirstAiStudio\src\app-studio`) has no Project concept at all today — creating an
App (`CreateAppModal.tsx`) creates a bare `Apps` row and nothing else. Flow Studio
(`BizFirstAiStudio\src\flow-studio`) already has a full Project concept: a Projects list/search
screen, a Project entity, and a "Design" action per project that opens the thing you actually
build inside it.

Binoy's explicit ask: App Studio should gain a Projects Search screen with a "Design App" button
per project, reusing Flow Studio's real Edit-Project and Edit-App-equivalent components rather
than building new ones — "why we should create duplicate code, I hate redundant code." If those
components aren't already in a reusable package, extract them into one first, then consume from
both products. The end state: users create a Project from either Flow Studio or App Studio and
get a unified experience, with "Design" opening the right tool for what that project actually is.

## Prior analysis (read this first — a research pass already answered the open questions)

A research-only fork already investigated this exact question against the live codebase and
backend today (2026-08-25) — do not re-derive any of the following from scratch:

1. **The real, live screen is `UnifiedDashboard.tsx`**, not `ProjectsDashboard.tsx` —
   `flow-studio-designer/src/components/Dashboard/ProjectsDashboard.tsx` is dead code (zero
   references anywhere). The screen actually wired into the app
   (`flow-studio-designer/src/WorkflowApp.tsx:450`) is `UnifiedDashboard.tsx`. Target this file,
   not the dead one.
2. **There is no search box today, despite the name.** `UnifiedDashboard` only has client-side
   pagination (6/12/18/24 per page, `localStorage`-persisted), no search/filter input. A real
   search box is new work in either product — don't assume one exists to lift as-is.
3. **The real hierarchy is Project → Process → ProcessThread, not Project → App 1:1.**
   `UnifiedDashboard`'s `DashboardView` is a 3-level drill-down. The "Design" button on a Project
   card (`handleProjectDesignClick`) is a *shortcut*: it fetches the project's first Process, then
   that process's first ProcessThread, and jumps straight to the flow canvas. It does not mean a
   Project has exactly one designable thing. App Studio's App concept is flatter (one App = one
   designable thing) — App Studio's own "Design" button should jump straight to the App, skipping
   the Process/Thread hop Flow Studio needs.
4. **The backend link between Project and App already exists, confirmed live, no schema work
   needed.** Two independent confirmations, already done — don't re-verify from scratch:
   - `flow-studio-api`'s `studioProjectApiClient.createWithStructure`
     (`clients/studioProjectApiClient.ts`, called from `ProjectForm.tsx:100`) creates
     Project + App + Process + ProcessThread in **one backend call** and returns `AppID`
     (`types/studioProject.types.ts:28`).
   - Queried App Studio's own live `Apps` table directly: `POST /api/v1/app-studio/apps/get-by-id
     {"id":1052}` returns a real `"projectID":null` column on every App row today — present,
     just unset. App Studio's frontend `AppRecord` type
     (`app-studio-api-client-js/src/types.ts:3-18`) doesn't declare `projectID` explicitly (it
     round-trips through a `[key:string]:any` index signature instead) — add it explicitly as part
     of this target, don't leave it invisible in the type system.
   - `Project_Projects` (`flow-studio-api/src/types/project.types.ts`) is a full standard BizFirst
     entity (TenantID/Deleted/Archived/ResID/SourceAppID FK, etc.) — already architecturally meant
     to be a shared, reusable concept, just only consumed by Flow Studio's UI today.
5. **Components are not reusable as a straight import — real extraction work, not a `link:` and
   done.** `UnifiedDashboard.tsx`/`ProjectForm.tsx` live inside `flow-studio-designer`, a
   monolithic package (reactflow, HIL chat/UI, atlas-forms designer/player, flow-observer — the
   entire canvas stack as transitive deps). Concrete couplings found, all fixable the same way
   this session already fixed `DesignerToolbar.tsx`'s `import.meta.env` coupling (take as props
   instead of reaching for global/store state directly):
   - `useUIStore`/`useAuthStore` from `@flow-studio/store` (theme/toast/auth, Flow-Studio-specific,
     not swappable via props today)
   - `@flow-studio/api`'s `projectApiClient`/`processApiClient`/`processThreadApiClient`
     (hardcoded imports, not injected)
   - Hardcoded inline styles using Flow Studio's own palette (`#020617`, `#3A8C45`) — no
     `@bizfirst/common-themes` token usage, unlike App Studio's own `ui/` primitives
   - `Modal`/`ConfirmDialog` from local `../Support/`, not the shared `@bizfirst/common-themes`
     modal classes App Studio's own `ui/Modal.tsx` already wraps
   - `ProjectForm.tsx` calls `studioProjectApiClient.createWithStructure` directly, hardcoding
     Process/ProcessThread creation as part of "create a project" — needs a mode/flag to create
     Project+App only when the host doesn't want a throwaway Process/Thread
6. **No existing shared "Project"/"Workspace" abstraction anywhere else** (`bizfirst-common`,
   `bizfirst-global` checked) — Flow Studio's is the only real implementation. This target is the
   first real cross-product consumer.
7. **Recommendation on "Edit App": do NOT reuse Flow Studio's Process/Thread edit form.** Flow
   Studio has no single "Edit App"-shaped component — its closest analog (`ProcessForm`/
   `ProcessThreadForm`) edits a Process/Thread, a different entity shape than an App. App Studio
   should keep its own existing app-metadata editing. Only the Project-level pieces (list screen,
   create/edit Project form) are the actual reuse target.

## Phase 0 — verify assumptions (fast, before building anything)

- Confirm with whoever owns the backend schema that `projectID` on `Apps` and Flow Studio's
  `Project_Projects.ProjectID` are the same concept, not a coincidental same-named column.
- Clarify cardinality: can one Project have multiple Apps over time (after creation), or is
  `createWithStructure`'s 1:1:1:1 shape enforced anywhere beyond creation-time convenience? Nothing
  in the current types enforces it either way — decide the intended contract before building UI
  that assumes one or the other.

## Phase 1 — extract the reusable pieces into a shared package

Create `bizfirst-common/project-management-react` (or similar — match this session's own `link:`
+ relative-path-workspace-member convention, see `app-studio/pnpm-workspace.yaml`'s comments and
`widget-handlers-form-widget`'s `DevelopmentHistoryLog.md` 2026-08-20 entry for the proven recipe;
do NOT use a `packages/*` glob, already found to pull in unrelated broken transitive deps).

Pull out of `flow-studio-designer` and decouple:
- The Project-list/grid view from `UnifiedDashboard.tsx` (not the Process/Thread drill-down layers
  — those stay Flow-Studio-specific) as a standalone, host-configurable component.
- `ProjectForm.tsx` (create/edit Project) — add a mode/flag to skip Process/Thread creation.
- The `EntityCard` presentational component `UnifiedDashboard` uses for project cards.

Decoupling requirements (apply the exact pattern already proven today in
`DesignerToolbar.tsx`/`EditWidgetModal.tsx`):
- Inject the API client via props/context, not a direct `@flow-studio/api` import.
- Inject theme/toast handling via props, not `@flow-studio/store`'s `useUIStore`.
- Swap the hardcoded flow-studio palette for `@bizfirst/common-themes` CSS custom properties (or
  make every color a themeable prop with the current values as defaults) — matches every other
  component this session touched.
- Use the shared `Modal` pattern (or accept one via props) instead of `../Support/Modal`.
- A `onDesignClick(project)` callback prop instead of a hardcoded Process/Thread-drill navigation —
  the host (Flow Studio or App Studio) decides what "Design" does.

Update Flow Studio's own `UnifiedDashboard.tsx`/`ProjectForm.tsx` to consume the new shared
package instead of their old local implementation (no duplicated logic left behind — verify with
a diff, not just "it still compiles").

## Phase 2 — App Studio integration

- Add `projectID?: number` to `AppRecord` (`app-studio-api-client-js/src/types.ts`) — real,
  typed, not riding the `[key:string]:any` index signature.
- Build App Studio's own Projects Search screen (a real search input this time — genuinely new
  work, not reuse, since upstream doesn't have one either) on top of the Phase 1 package.
- Wire "Design App" to open App Studio Designer directly against that project's App —
  App Studio's flat model means no Process/Thread hop is needed, unlike Flow Studio's own button.
- Decide (with Binoy, this is a real product decision, not an implementation detail): does
  creating a new App still work standalone (current behavior), does it now require
  picking-or-creating a Project first, or is Project optional-but-encouraged? Binoy's framing
  ("Ideally we should Open a project in the beginning of the app rather than creating an app")
  leans toward Project-first, matching Flow Studio's own flow — implement that unless told
  otherwise, but flag the tradeoff (existing apps have `projectID: null`; decide the fallback UX
  for those) rather than silently picking one.

## Phase 3 — Edit App

Per the prior-analysis recommendation: do NOT force-fit Flow Studio's Process/Thread edit form.
Keep/extend App Studio's own existing app-metadata editing. If it doesn't already live in a
reusable spot, that's a separate, much smaller cleanup — not blocked on this target.

## Constraints (apply throughout)

- No auto-commits/pushes (repo-wide policy).
- No premature abstraction — the shared package should hold exactly what's proven reused by two
  real consumers (Flow Studio + App Studio) by the end of this target, not speculative extra
  flexibility for hypothetical future products.
- `pnpm typecheck`/`pnpm build` clean for every touched package/app before considering a phase
  done. `apps/app-studio-designer` and `apps/app-player` were both confirmed building 100% clean
  earlier the same day (2026-08-25) after a separate build-blocker fix pass — do not regress that.
- Dated `DevelopmentHistoryLog.md` entries in every touched package (create if missing).
- `app-studio/docs/QA-Session-2026-08-25.md` has the full context for what else changed in App
  Studio the same day — skim it so nothing here collides with or duplicates that work.
