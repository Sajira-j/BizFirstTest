# Wix-Style Design Series — Implementation Status

**Purpose of this file**: single source of truth for where this multi-task build stands. If this
session drops, read this file first — it tells you what's done, what's running, what's next, and
*why* the sequencing is what it is, so you don't have to re-derive it from the six design docs.

Last updated: 2026-08-30 (mid-session, Wave 1 in progress — no completions yet).

## Wave 1 live status (checked via ListAgents, not assumed)

All four Wave 1 agents are RUNNING, none have reported completion:

| Agent | Task | Status as of last check |
|---|---|---|
| `a9e8bbb8032abf21d` | Task 1 (digital-assets-library, design+build) | **✅ COMPLETE** (reported, not yet independently live-verified by the coordinator — see below) |
| `aa0353b8ba1d21e27` | Task 2 (click-to-edit unified Designer) | **✅ ALL 6 PHASES COMPLETE**, agent-verified; coordinator's own independent verification pending (see below) |
| `a6544d0100e9ec18c` | Task 5 Phase 1 (undo/redo) | **✅ COMPLETE, but needs a follow-up live re-verification** (see below) |
| `ad256464424c102a1` | Task 4a (SEO fields + checklist + app-wide tab) | **✅ COMPLETE, one visual re-verification pending** (see below) |

### Task 1 — COMPLETE, agent's self-report

- **Serving model (coordinator's correction, confirmed applied)**: public assets get an opaque
  `PublicAssetKey` (`Guid.NewGuid("N")`-shaped, matching the existing `Platform.StorageServers.
  PresignService`'s own key convention) minted once at publish time; the public `FileUrl` resolves
  directly against a public-read bucket with zero DB lookup / zero app-server relay per view.
  `DocumentID` never appears in the public URL. Private assets go through ACL check → short-lived
  presigned URL. The old `/api/v1/documents/{id}/download` DocumentID-keyed pattern is explicitly
  called out as pre-existing tech debt this app deliberately does not extend.
- **Built**: new `apps/digital-assets-library` (port 6111, thin shell); new `@doc-app/react/src/assets/*`
  components (`AssetGrid`/`AssetCard`/`AssetPreview`/`AssetUploadPanel`/`AssetVisibilityBadge`, ~70%
  reuse of existing `@doc-app/react`/`@doc-app/api-client`); additive `DocumentClient` methods against
  the corrected contract; two idempotent, NOT-executed SQL migrations; port-registry + task-runner
  entries; `doc-app/DevelopmentHistoryLog.md` updated. No git commits.
- **Explicitly NOT built this pass (agent's own flag, not glossed over)**: the backend publish/
  unpublish/presigned-view endpoints, and which concrete service backs `IDocumentStorageProvider`
  today. **Every asset behaves as private until that backend lands — public serving is not yet
  functionally testable end-to-end**, only the frontend shell/component layer is.
- **Coordinator's independent verification (2026-08-30)**:
  - `pnpm install` picks up the new app + packages correctly (workspace grew from 4 to 9 projects).
  - Dev server boots clean on port 6111, zero console errors.
  - Live browser test: unauthenticated visit correctly redirects to the real Passport login
    (`localhost:8001`) — proper auth-gating for a private-asset app, not a bug. Could not proceed
    past login: entering a password into any field is a hard rule this session does not override
    regardless of Binoy's Chrome-access permission grant (that covers port/tab access, not
    credentials). **Full authenticated E2E (upload/browse/visibility-toggle UI) is blocked on Binoy
    logging in himself, or providing a dev/fake-auth bypass if one exists** — flagging this back
    rather than working around it.
  - `pnpm build`'s `tsc --noEmit` step fails — but verified this is **pre-existing, workspace-wide
    tech debt**, not a regression: `document-manager` (completely unmodified) fails with the
    identical error set (`Cannot find module '@passport/api-types'/'@passport/config'/
    '@passport/constants'`). Isolated the new code specifically (`@doc-app/api-client`,
    `@doc-app/react`, `digital-assets-library` app, filtering out only the known baseline errors):
    **zero new type errors**. The new code itself is clean; the failure is inherited, not introduced.

### Backend fix (upload IsPublicAsset / publish-unpublish / presigned-view) — mostly done (2026-08-30)

- **Storage provider gap bigger than scoped**: `Platform.StorageServers`/`PresignService` has NO
  public-read/ACL capability at all (confirmed via grep, zero hits). Built an interim
  `LocalDocumentStorageProvider` (local disk) + a new anonymous opaque-key-keyed controller instead
  of blocking on the missing capability — flagged as a swappable interim, not silently substituted.
- **Publish/unpublish**: `Document.cs` + repository/service layers (`PublishAssetAsync`/
  `UnpublishAssetAsync`/`GetPublishedAssetByKeyAsync`) + `BaseDocumentController`'s `publish-public`/
  `unpublish-public`/`presigned-view-url` endpoints — **live-verified working**: real file published
  as public, DB row confirmed with correct opaque key (not DocumentID), fresh key mints on
  re-publish. Full solution rebuild: 0 errors.
- **404 bug: root cause found and FIXED (2026-08-30), TASK 1 BACKEND NOW COMPLETE**. Real cause
  (confirmed via a live route-table dump, not a guess): `PublicAssetController.cs` was created only
  in the `.Api` project and never copied into the platform WebApi project
  (`BizFirst.Ai.Platform.Web.Server.Core\Controllers\...`), which is this codebase's own established
  convention (never `ProjectReference` a domain's `.Api` project — copy the controller source file
  instead, same as `DocumentController.cs`/`PublicShareController.cs` already do). Its routes were
  therefore completely absent from the running app's route table regardless of rebuilds. The
  SPA-fallback-middleware hypothesis was reasonable but ruled out directly (no `UseStaticFiles`/
  `MapFallback`/`UseSpa` exists in this pipeline). Fixed by copying the controller file, matching the
  existing convention exactly. Full solution rebuild: 0 errors.
- **Environmental note, also fixed**: the machine had 0.5GB free RAM with 12 concurrent `dotnet`
  processes — 11 were leftover MSBuild build-server nodes (`nodeReuse:true`) from repeated rebuild
  cycles, not needed once idle. Coordinator killed all 11 (safe — idle build caches, not the running
  WebApi), reclaiming ~350MB and stabilizing the WebApi enough to actually respond.
- **Coordinator's own live verification (2026-08-30), full end-to-end**: found the real route
  (`api/v1/public/assets/{publicAssetKey}`, from `BasePublicAssetController.cs`) and curled the
  already-published test asset (DocumentID 9, key `31d5790b41fe4cb2945bb9f1a093fa1b`) directly:
  **HTTP 200, 56 bytes served, `content-type: application/octet-stream`, zero auth header required**
  (genuinely anonymous/public) — and the URL contains ONLY the opaque key, no `DocumentID` anywhere.
  **This is full, real, end-to-end confirmation the public-asset serving model works correctly.**
  Task 1 (digital-assets-library) is now functionally complete: upload honors visibility,
  publish/unpublish works, private presigned-view exists, and public assets genuinely serve bytes to
  anonymous requests via a non-enumerable key.

### Task 5 Phase 1 — COMPLETE, agent's self-report + a real process finding

- **Shipped**: `undoStack`/`redoStack` + `mutateLayoutWithUndo`/`undo`/`redo` in `app-selection.store.ts`
  via immer `produceWithPatches`/`applyPatches`; `moveSection` and the section-style `patchSection`
  path route through it; toolbar undo/redo buttons (disabled-state, tooltip labels); Ctrl+Z/
  Ctrl+Shift+Z/Ctrl+Y keyboard handler. Added `immer` to `app-studio`'s OWN nested pnpm-workspace
  catalog (it only existed in the top-level umbrella one — real, correctly-diagnosed gap). Category B
  actions (delete/reorder/rename/widget-config-save) deliberately not covered, per design doc scope.
  No git commits.
- **Real collision, handled correctly**: Task 2 (running concurrently) extracted `SectionDetailsContent`
  out of `CanvasPanel.tsx` mid-flight — exactly the kind of overlap flagged as a risk going in. Task 5's
  agent verified its own `patchSection` fix survived that extraction correctly in the relocated
  `details/SectionDetailsContent.tsx`. Confirms the "run in parallel, expect Task 5 to finish first,
  verify after" call was the right one — but it was closer than planned.
- **Process finding — fix before dispatching Wave 2/3 in parallel**: live-verification was
  INCONCLUSIVE, for two reasons: (1) hit a frozen tab from Task 2's own transient broken bundle
  mid-edit, and (2) a second attempt showed signs of the Task 2 agent interactively using the SAME
  shared browser tab concurrently (a Widget Details panel open that this agent never triggered).
  **This is the exact "agents bombarding each other" risk Binoy flagged, showing up at the
  live-testing layer, not just the code-editing layer** — parallel agents were sharing one Chrome tab
  for live verification. **Fix for future waves: every agent doing live browser verification must
  create its OWN tab via `tabs_create_mcp` rather than reusing whatever tab is already open,
  whenever more than one agent may be live-testing concurrently.** Will add this explicitly to every
  future dispatch prompt.
- **Confirmed instead, via screenshot**: toolbar buttons render correctly; Layout Design mode
  correctly loads qoboto's real 11-section layout.
- **Outstanding**: a real click-test (move section → Ctrl+Z → Ctrl+Shift+Z, confirm canvas + live
  preview both reflect it) once Task 2 lands and a dedicated tab is available — coordinator will run
  this personally as part of the full Wave 1 E2E pass (see Testing Plan below), not re-delegate it.

### Task 4a — COMPLETE, agent's self-report

- **Shipped**: `AppPageSeo` type + additive `AppPageRecord.seo?` field; backend migration
  `V018_UP/DOWN_AIExt_AppPages_SEOConfiguration.sql` (NOT executed, left for review) + matching
  `AppPage.cs` entity property (Domain project builds clean); fallback-chain title/meta-tag wiring
  (`seoMeta.ts`/`useDocumentSeoMeta`) into `AppPlayer.tsx`, gated on `!studioMode`; full 10-row SEO
  checklist (`computeSeoChecklist.ts`, DOMParser-based, real content-widget HTML via the existing
  `ContentSanitizer`) + the "SEO & Analytics" 6th tab in `AppConfigScreen.tsx`; honest analytics
  empty state (no fake data). `AppState`/`LoadedAppData` gained `name`/`appCode`/`meta` (were being
  silently discarded before this pass). All 5 touched frontend packages + backend Domain project
  build/typecheck clean. `DevelopmentHistoryLog.md` entries in all 5 packages + the AppStudio DB
  migrations log. Did NOT touch `CanvasPanel.tsx`/`DesignerToolbar.tsx` (correctly stayed out of
  Task 2/5's files). No git commits.
- **Live-confirmed win**: qoboto's `app-player` browser tab title now correctly reads
  "Qoboto - Decentralized Website Builder" — a real, previously-missing capability (zero
  `document.title`/meta-tag handling existed in `app-player` before this).
- **Verification gap, cause identified (not a code bug)**: the new SEO tab's own visual render
  wasn't confirmed — `/api/v1/app-studio/widgets` hung under this session's concurrent multi-fork
  load at verification time. Independently confirmed via direct curl: `/pages` returned in 155ms,
  `/widgets` timed out at 25s — **this is the SAME recurring Consolidated WebApi
  memory-pressure/stall issue** already root-caused earlier this session (the "App #1526" toolbar
  bug investigation) — a shared-backend load problem, not a bug in this new code. Worth noting for
  the coordinator's own E2E pass: running 3-4 agents' live-verification concurrently against the
  same backend process measurably increases how often it stalls; may need to stagger or restart the
  WebApi before an intensive verification pass.

### Task 2 — Phases 1-2/6 COMPLETE, agent's self-report; resumed for Phase 3-6

- **Shipped and live-verified**: Phase 1 (overlay shell via `@floating-ui/react`, replacing the docked
  320px panel — this is also what relocates `WidgetDetailsContent`/`SectionDetailsContent`, the thing
  Task 3's own doc said it needs) and Phase 2 (inline Tiptap landing view for content widgets,
  save-on-dismiss). Both `tsc --noEmit` clean. Save-on-dismiss verified genuinely working via a direct
  backend query, not just a visual check.
- **Open question resolutions**: dismiss-to-save implemented as the doc recommended (no Save button
  in the landing view, `saveIfDirty` ref fires only on actual change). Site Structure drawer
  reachability question still open — not yet reached (that's Phase 4).
- **Deliberate checkpoint before Phase 3**: Phase 3 (unify Pages + Layout into one canvas, retire
  `PageCanvas.tsx`, touch `AppShell.tsx`'s core routing) is the biggest, riskiest step in the whole
  design doc — the agent stopped here rather than rush it while the backend was under heavy
  concurrent load from Task 4a/5 running at the same time. Good judgment call, matches this
  codebase's own "live-verify before moving on" convention.
- **Two real cross-agent reconciliations handled correctly**: Task 5's undo/redo landing mid-edit, and
  a Tiptap HMR artifact — both resolved, logged in `02-click-to-edit-inline-designer.md`'s own "Build
  Progress" section.
- **Coordinator action**: resumed the agent through Phases 3-6 — ALL COMPLETE as of this update.
  Phase 3: `CanvasPanel.tsx` rewritten into one continuous canvas, `PageCanvas.tsx` deleted,
  `AppShell.tsx` simplified from 4 body branches to 2. Phase 4: shared `Drawer.tsx` primitive; Pages
  CRUD + `SiteStructurePanel.tsx` rehosted as drawers; new shared `useAppPages` hook (avoids a
  duplicate-Home-page race). Phase 5: in-canvas "+ Add Section" button, deliberately scoped down from
  the fuller hover-affordance vision to avoid touching the shared `AppPlayer.tsx` rendering package
  production `app-player` also depends on — a good, conservative call. Phase 6: page-level URL routing
  `/{appCode}/page/{slug}`, both push and cold-deep-link directions verified.
- **4 rounds of live feedback from Binoy, all fixed and verified**: fullscreen toggle added once to
  shared `Modal`/`OverlayPanel`/`Drawer` primitives (not per-instance); a real Site Structure drawer
  transparency bug; a real stacking-context bug where `AddWidgetModal` rendered clipped inside
  `OverlayPanel` (fixed via `createPortal`); a follow-up z-index bug from the same root cause, fixed
  properly with a new shared `zIndexScale.ts` instead of more ad hoc numbers.
- **Verification**: `pnpm tsc --noEmit` clean across all 3 touched packages
  (`app-studio-designer-components-react`, `app-studio-store-react`, `app-studio-designer`). Two real
  dev-environment gotchas correctly diagnosed as environmental (not code regressions): transient
  backend contention under multi-agent load, and a stale-Vite-HMR-state tab crash.
- **Coordinator's own live-verification (2026-08-30) found a REAL regression**: fresh-tab load of
  qoboto (`/qoboto/page/home`) got permanently stuck showing "Loading…" inside the canvas with
  "No engine active" at the bottom, even after all underlying app-data API calls (by-code, get-by-id,
  pages, widgets) resolved 200. Confirmed via `read_console_messages`/`read_network_requests`, not
  assumed. Root-caused precisely: `LivePreviewPanel`'s own loading text is "Preparing preview…"
  (centered) — NOT what was showing — meaning `engine` was already non-null and `AppPlayer` itself
  was rendering but stuck with `engine.app.status` never leaving `'loading'`.
- **Root cause, fixed**: `fetchJson()` (`app-studio-api-client-js/src/http.client.ts`), used by every
  app-studio API call, had **zero client-side timeout**. When the backend stalls (this dev box is
  genuinely memory-starved — confirmed 0.39GB free of 7.72GB total — Binoy's own diagnosis, correct),
  the underlying `fetch()` hangs forever, `StoreBackedAppDataLoader.loadApp()` never resolves or
  rejects, and the engine sits at `'loading'` permanently with zero error surfaced anywhere. Fixed by
  adding `AbortSignal.timeout(30_000)`, matching the existing `HttpClient` convention elsewhere in this
  codebase — systemic (every call gets it), not a narrow patch. Also fixed the `UndoSelection.widgetId`
  → `widgetID` casing finding from the code review (3 call sites).
- **Verification status**: all 4 touched/downstream packages typecheck clean. **NOT yet live-verified
  with a real, fully-loaded qoboto layout** — the fix agent's browser sessions both lost real auth
  mid-investigation and it correctly refused to touch the resulting real login form (autofilled
  credentials) rather than force a fresh-tab test. Coordinator is also holding off on live-verifying
  this personally right now given the confirmed severe memory pressure on this machine — will verify
  once memory eases rather than adding more concurrent load.
- **Do NOT dispatch Wave 2 (Task 6/4b) yet** — Task 2 needs this fix live-verified with real data
  first. This is now the single blocking item before Wave 2.

No other wave item is running. Do not treat anything in this table as done until a completion
notification is received and independently live-verified — update it only from confirmed agent
reports, never from elapsed time or assumption.

## Testing plan (per Binoy's instruction — E2E only after builds land, not mid-flight)

Once Task 1 (digital-assets-library) reports completion: functionally test the new app itself (upload/
download/admin flow, public vs private asset serving via the corrected opaque-URL scheme, not
DocumentID-based).

Once Wave 1's `app-studio` changes (Tasks 2, 4a, 5) report completion: run a full E2E pass of the
Designer against the qoboto app (AppID 1526, AppCode "qoboto") covering all three shipped changes
together in one pass (not each in isolation) specifically BECAUSE they landed concurrently and touched
adjacent files (`DesignerToolbar.tsx` especially) — this is exactly the kind of interaction bug that
only shows up once everything is actually integrated, not from any single task's own live-verification
during its build.

Binoy has pre-authorized Chrome/claude-in-chrome access and testing any localhost port for this —
no permission prompts needed for either testing pass.

## The six tasks

| # | Task | Design doc | Design status | Build status |
|---|---|---|---|---|
| 1 | Digital Assets Library | `01-digital-assets-library-design.md` | In progress (agent revising after asset-URL correction) | In progress — design + build combined per Binoy's explicit choice |
| 2 | Click-to-edit inline Designer (no mode-switching) | `02-click-to-edit-inline-designer.md` | ✅ Done (~13-18 eng-days) | **Dispatching now (Wave 1)** |
| 3 | Free-form drag-and-drop positioning | `03-freeform-drag-drop-positioning-analysis.md` | ✅ Done (~20-27 dev-days) | Not started — waits on Task 6 (Wave 3) |
| 4 | SEO checklist + analytics panel | `04-seo-analytics-panel-design.md` | ✅ Done | Split: 4a dispatching now (Wave 1), 4b waits on Task 2 (Wave 2), 4c (analytics backend) explicitly out of this batch |
| 5 | Undo/redo | `05-undo-redo-design.md` | ✅ Done | Phase 1 **dispatching now (Wave 1)**; Phase 2 out of this batch |
| 6 | Responsive breakpoint editing | `06-responsive-breakpoint-editing-design.md` | ✅ Done (~2.5-3.5 days) | Not started — waits on Task 2 (Wave 2) |

## Why this sequencing (not arbitrary — grounded in the docs themselves)

Binoy's instruction: avoid shared-file collisions between agents coding in parallel. The
consolidated review of all 5 completed docs found real, concrete overlap:

- **Task 2 relocates `SectionDetailsContent`/`WidgetDetailsContent`** out of `CanvasPanel.tsx`
  (which it retires) into a new overlay shell. **Task 3's own doc** says its Flow/Canvas mode-switch
  toggle belongs "in the new Section Details panel this session shipped" — i.e. it must target
  wherever Task 2 relocates that panel to, not the soon-to-be-retired docked one. **Task 2 must ship
  before Task 3.**
- **Task 3's own doc** ("Blockers/risks") and **Task 6's own doc** (section 7, "Interaction with
  Task 3") *independently* both recommend the same thing: agree the `{desktop, tablet?, mobile?}`
  coordinate/style schema shape and ship Task 6 before Task 3, so Task 3 doesn't do a breaking data
  migration later. **Task 6 must ship before Task 3.**
- Both of the above converge on the same order: **2 → 6 → 3**. Not designed that way on purpose by
  each doc's author (they were written by independent agents, unaware of each other's conclusions)
  — it fell out of the actual technical dependencies, which is a good sign it's the real order, not
  a guess.
- **Task 2 and Task 5 both touch `DesignerToolbar.tsx`** (Task 2 removes Pages/Layout Design
  buttons; Task 5 adds undo/redo buttons) — different regions of the same file. Task 5's Phase 1 is
  small (2-3 days) vs. Task 2's DesignerToolbar changes landing mid-plan (~day 8-10 of 13-18) — so
  Task 5 is dispatched **in parallel with, but expected to finish before**, Task 2 reaches its own
  toolbar work. Lower collision risk than the reverse order.
- **Task 4** splits cleanly: the app-wide "SEO & Analytics" tab (`AppConfigScreen.tsx`) has zero
  dependency on anything else here — safe to build now. The **per-page** SEO accordion explicitly
  needs Task 2's per-page details panel to exist first (the design doc says so itself, section 6).
- **Task 1** lives in a completely separate monorepo (`doc-app`, not `app-studio`) — zero file
  overlap with anything else in this batch, safe to run fully in parallel throughout.

## Undo/redo CRITICAL corruption bug — FIXED (2026-08-30)

Coordinator fixed directly (no agent dispatch needed — small, well-understood store-layer change):
`removeAppWidget`/`moveAppWidget`/`reorderWidgetsInSection`/`removeSection`/`renameSection` (all
Category B, untracked) now clear `undoStack`/`redoStack` in their own `set()` calls, closing the
stale-patch-replay corruption path the code review found. `tsc --noEmit` clean. Full writeup in
`app-studio-store-react/DevelopmentHistoryLog.md`. Live verification still pending (same
memory-constrained-machine hold as Task 2's fetch-timeout fix).

## Task 8 (new, approved 2026-08-30) — "View" menu: Always Show Labels toggle

Approved design (Binoy: "i go with your recommendation"): ONE toggle, not two. A "View" top menu
with a single "Always Show Labels" item (off by default). Default behavior is hover-only reveal for
BOTH section/widget name labels AND the "..." action-button affordance (no separate toggle for the
action button — folds into the same hover/selection reveal, matching Wix/Webflow/Squarespace/Framer
convention). "Always Show Labels" is the power-user escape hatch (matches Webflow's actual "element
outlines" toggle) for seeing a whole complex layout's structure without hovering each element.
Retains 100% of Task 2's existing click-to-edit inline behavior — hover only changes chrome
visibility, never interaction. Not yet built — queued behind current fixes/live-verification, small
enough to build directly once memory eases (no design doc needed, scope is already fully specified
here).

## Task 9 (new, built 2026-08-30) — Widget Toolbox + Cross-Section Drag-and-Drop

Not one of the original nine wix-style-design tasks — a new post-batch feature request (Binoy: "is
widget be dragged from one section to another? also a widget can be dragged from a toolbox? In the
toolbox, i like to see new widgets and existing widgets as toolbox items?" ... "Spec it out as a
design doc and complete the coding for this"). Design doc + build combined, same pattern as Task 1.

- **Design doc**: `07-widget-toolbox-and-cross-section-dnd.md` — covers the new "Toolbox" drawer
  (drag new/existing widget types onto a section), cross-section widget-chip dragging (previously
  impossible — `reorderWidgetsInSection` only ever operated within one section, and each section's
  own drag state was local `useState`, invisible across sections), the four now-distinct drag
  interactions in this UI and how a user tells them apart, and why native HTML5 drag-and-drop
  (never `@dnd-kit`) is the right call here too.
- **Built**: `canvas/dragPayload.ts` (shared native-DnD payload contract), `widgets/addWidgetActions.ts`
  (widget-creation logic extracted from `AddWidgetModal.tsx`, now shared with the Toolbox — no
  duplication), `toolbox/ToolboxPanel.tsx` (new drawer), a new `moveAppWidgetToSection` store action
  (`app-selection.store.ts`, follows the file's own Category B/undo-clearing/`structuralGeneration`
  convention), and drop-zone wiring on both existing `WidgetChip`-list surfaces (Site Structure
  drawer's `SectionCard.tsx`, Section Details popover's `SectionDetailsContent.tsx`) plus the live
  WYSIWYG canvas itself (`LivePreviewPanel.tsx`/`CanvasPanel.tsx`, via the same `[data-section]`
  delegation the existing click-to-select handler already uses — zero `AppPlayer` changes).
  `AddWidgetModal.tsx`'s existing click-through flow is kept, unchanged in behavior (refactored
  internals only) — it's still the required path for `form`/`workflow-template`/
  `workflow-template-category` (hard-required config fields a bare drag can't safely supply).
- **Collision-safety honored**: re-read `WidgetTypeRegistry.ts`, `AddWidgetModal.tsx`, and
  `app-selection.store.ts` fresh immediately before each file's first edit (all three were flagged
  as concurrently in-flux) rather than trusting an earlier read — found `workflow-template-category`
  had already been promoted out of `NOT_YET_AVAILABLE` with a real handler package and a third
  hard-required config field by the concurrently-running widget-review-followup effort; built on
  top of that real state (added it to the "needs `AddWidgetModal`, not drag" set alongside
  `form`/`workflow-template`) rather than clobbering it.
- **Verification**: `pnpm --filter @app-studio/designer-components-react run typecheck` and
  `pnpm --filter @app-studio/store-react run typecheck` both clean; a full `pnpm -r typecheck`
  sweep across all 54 workspace projects also clean (exit 0), specifically to catch any downstream
  consumer of the changed `WidgetChip`/`AddWidgetModal`/`LivePreviewPanel` prop signatures. No test
  infra exists in either touched package (confirmed before starting, not introduced as a side
  effect). **Live browser E2E verification is explicitly NOT done this pass** — same "build now,
  verify everything together later" tracking convention as every other feature in this initiative.
  No git commits.

## Round 2 remaining fixes — coordinator applying directly (2026-08-30)

- **MEDIUM, FIXED**: `SeoAnalyticsTab.tsx` canonical URL now calls the shared `computeCanonicalUrl`
  (exported from `@app-studio/generic`) instead of reimplementing it — one source of truth with the
  real render path.
- **MEDIUM, FIXED**: `SeoAnalyticsTab.tsx` content-source bug — now reads `aw.configuration` (the
  real per-page override) instead of the shared catalogue template, falling back only when a
  placement has no override of its own.
- **LOW/PLAUSIBLE, FIXED (defense-in-depth)**: `assetEmbed.ts` gained a URL-scheme allowlist
  (`isSafeEmbedUrl`) — rejects anything that isn't absolute http(s)/protocol-relative/root-relative,
  returns `''` (safe no-op) otherwise. `BackgroundTab.tsx`'s raw `url('...')` splice (2 call sites)
  now escapes backslash + single-quote via a new `escapeCssUrlLiteral` helper, rather than relying
  entirely on a downstream guard in a different package.
- **LOW, HOLDING**: `reorderCanvasWidgetZIndex`'s non-atomic swap — not yet fixed, deliberately
  holding because `app-selection.store.ts` is being actively edited right now by the concurrently-
  running reload-performance-fix agent (new `structuralGeneration`/`lastWidgetSave` fields already
  visible in that file) — will fix once that agent reports back, to avoid a real collision.
- A THIRD independent review is also running now (user-requested), broadened to cover `app-player`
  (the real end-user runtime, least-scrutinized so far), `document-manager`/`knowledge-app` (sibling
  apps sharing `@doc-app/react` with the new digital-assets-library work), the backend
  `PublicAssetController` under adversarial input, and a rigorous trace of the two items round 2
  marked PLAUSIBLE-not-fully-verified (Task 3 nested-section overlap, assetEmbed/BackgroundTab
  reachability claims).

## THIRD CODE REVIEW COMPLETE, broadened to all apps — 1 CRITICAL (fixed), rest still open (2026-08-30)

The third independent review (dispatched per Binoy: "you could also do one more critical review
wusing a bg agent. do thorough review. also review all apps") was explicitly tasked to re-verify
rounds 1 and 2's fixes rather than trust them, and to cover `app-player`, `document-manager`,
`knowledge-app`, digital-assets-library's backend, and `atlas-forms`, not just the wix-style-design
diff.

- **CRITICAL, FIXED**: `cssInjector.ts`'s round-1/round-2 blocklist guards
  (`sanitizeScopedCss`/`declBlock`'s per-value check) were themselves bypassable via CSS hex-escape
  sequences (`\7B` decodes to `{`, `\40` to `@`, per CSS Syntax Module Level 3 §4.3.7) — the review
  supplied a passing exploit (`red\7D  body\7B display:none!important\7D `) containing zero literal
  brace/at characters. Fixed: new `decodeCssHexEscapes()` decodes before the existing blocklist
  runs, on both `sanitizeScopedCss` and the `declBlock` value guard (`isUnsafeCssValue` inlined into
  `declBlock` since decoding and the emitted value now need to be computed together); the decoded
  form is what gets emitted, not the original escaped text. Round-2's key guard
  (`isUnsafeCssKey`'s allowlist) was re-verified as NOT vulnerable to this — a backslash was never
  in that allowlist. 5 new regression tests (25/25 total pass), `tsc --noEmit` clean. Full writeup
  in `app-handlers-generic/DevelopmentHistoryLog.md`.
- **Verified CORRECT, no change needed**: round 2's `isUnsafeCssKey` allowlist (immune to the
  hex-escape class by construction); the pointer-listener `useCallback`+ref fix in
  `LivePreviewPanel.tsx` (genuinely stable across renders, confirmed still holding); the
  `updateWidgetPosition`/`reorderCanvasWidgetZIndex` undo-stack-clearing convention in
  `app-selection.store.ts`; `assetEmbed.ts`'s new `isSafeEmbedUrl` allowlist and
  `BackgroundTab.tsx`'s new `escapeCssUrlLiteral` — both confirmed immune to the hex-escape bypass
  class specifically because they operate inside a properly quote-delimited string context rather
  than an unquoted token splice; anonymous public-asset-key fetch re-confirmed opaque
  (no `DocumentID`/sequential ID in the URL, 200 + real bytes, zero auth header).
- **HIGH, FIXED**: canvas-mode (`layoutMode: 'canvas'`) absolute-positioned widgets overlapped a
  section's own nested child sections' rendered content — `AppSectionRenderer` put `position:
  relative` on the whole section root, so every widget's offsets resolved against the section's own
  top edge regardless of where its nested children rendered in flow. Moved `position: relative` onto
  the `[data-section-widgets]` wrapper instead, so the containing block starts after the nested
  sections' own block. 2 new regression tests (`AppPlayer.canvasLayout.test.tsx`), 48/48 tests pass,
  `tsc --noEmit` clean. Full writeup in `app-handlers-generic/DevelopmentHistoryLog.md`.
- **MEDIUM, FIXED**: `BreakpointService.override()`/`clearOverride()` (built for Task 6, explicitly
  documented in its own code as "designer-only preview toggle") had ZERO call sites anywhere in the
  codebase — never wired to the Task 6 device-preview toolbar, so section-level `hiddenOn`/style
  resolution only reacted to real `window.innerWidth`/resize, ignoring whatever device the Designer's
  own preview toolbar was set to. `LivePreviewPanel.tsx` now drives the real service from its device
  toggle (`override()` for desktop/tablet/mobile, `clearOverride()` for full-width and on cleanup).
  No automated test (this package has no test tooling at all) — covered by the planned E2E pass.
  `tsc --noEmit` clean. Full writeup in `app-studio-designer-components-react/DevelopmentHistoryLog.md`.
- **LOW, FIXED**: `LocalDocumentStorageProvider.GetSafePath`'s (C#, BizFirstPayrollV3) traversal
  guard had a missing-trailing-separator bug (bare `StartsWith` prefix check let a same-prefix
  SIBLING directory pass) — not currently reachable in practice, but the guard itself didn't hold
  on its own terms. Fixed with a separator-terminated prefix check; 3 new regression tests
  (`LocalDocumentStorageProviderTests.cs`, new — this also required fixing an unrelated
  pre-existing `KnowledgeControllerTests.cs` compile blocker to get the test project building at
  all; one further pre-existing, unrelated `InsertFromFile` test failure was surfaced and flagged,
  not fixed, as genuinely out of this pass's scope). Full writeup in `BizFirstPayrollV3`'s
  `Go/Documents/DevelopmentHistoryLog.md`.
- **LOW, FIXED**: `documentClient.ts` carried stale "not yet implemented" doc comments on 3 methods
  that are actually built and working (`publishAsPublicAsset`/`unpublishPublicAsset`/
  `getPresignedViewUrl`) — traced the real backend (`BaseDocumentController`, routed automatically
  through the platform's concrete `DocumentController`) and corrected the comments. Comment-only,
  `tsc --noEmit` clean.
- **LOW, still deliberately HOLDING**: `reorderCanvasWidgetZIndex`'s non-atomic swap — re-confirmed
  accurate but cosmetic; still holding on the reload-performance-fix agent's edits to
  `app-selection.store.ts` finishing first, to avoid a real collision.
- Reviewed and found clean, no findings: `document-manager`/`knowledge-app` (both share
  `@doc-app/react` with the new digital-assets-library work — no injection/auth gaps found in this
  pass), digital-assets-library backend under adversarial input beyond the asset-key opacity check
  above.

**Status after round 3**: ALL findings fixed (2026-08-30), including the deliberately deferred
`reorderCanvasWidgetZIndex` non-atomic swap — the reload-performance-fix agent reported back
(root-caused and live-verified fix, see its own report below) and its edits to
`app-selection.store.ts` landed cleanly, so the z-index fix was applied right after with no
collision. `tsc --noEmit` clean across every touched package. Given three consecutive review rounds
have each found a real issue the prior round missed, a fourth pass may be worth considering — not
yet requested by Binoy, so not started unprompted. The next step per his own "fix all issues and
test e2e" instruction is the live E2E pass, which has not yet happened this batch.

## Widget-edit full-canvas-reload — FIXED, live-verified (2026-08-30)

Real reported bug, dispatched as its own background agent, running concurrently with the round-3
review/fix work above: "When I edit a widget, the whole page is reloaded... can you make this a
background job? and a smooth ui update." Confirmed root cause (two layered issues, not one):
1. `useLivePreviewEngine.ts`'s reload effect keyed off `appWidgets`'s array *reference* — any
   content-only edit produced a new reference indistinguishable from a structural change.
2. `StoreBackedAppDataLoader.loadApp()` (what the heavy path re-ran) unconditionally re-fetches the
   ENTIRE tenant widget catalogue plus all app pages on every reload, regardless of what actually
   changed — the dominant, previously-undocumented cost.

Fix: new `structuralGeneration` counter (bumped only by genuinely structural actions —
add/remove/reorder widget or section, undo/redo) the heavy reload effect now keys off instead of
raw `appWidgets`; a new light-sync path (`lastWidgetSave`/`applyWidgetSaveLocally`,
`AppEngine.updateWidgetCatalogueEntry`/`updateAppWidgetRecord`) patches the running engine in place
for a content-only save — cheap re-render, zero network calls. `WidgetEditorFields.tsx`'s
`handleSave` now calls the light path instead of a full `refreshAppWidgets()`.

**Live-verified against Qoboto (AppID 1526)**: edited the Hero Heading widget via the real Tiptap
inline editor, dismissed to save. Network log showed exactly the widget's own two PUT calls — no
catalogue-list or pages-list re-fetch — canvas updated in place with no reload flash, rest of the
tree stayed mounted. Test edit reverted afterward, Qoboto left unchanged. `tsc --noEmit` clean
across all touched packages. Full writeup in each touched package's own `DevelopmentHistoryLog.md`.

## SECOND CODE REVIEW COMPLETE — caught 2 of round 1's own fixes still broken (2026-08-30)

A second independent review, specifically tasked with verifying round 1's 5 fixes rather than
trusting them, found:

- **CSS injection fix was INCOMPLETE** — round 1 only guarded property VALUES; property KEYS were
  still spliced raw, reopening the identical Critical vulnerability via a different vector (a
  direct authenticated AppWidget API call, no Designer UI needed). Fixed with a proper allowlist
  (`isUnsafeCssKey`, `/^[a-zA-Z][a-zA-Z0-9]*$/`) — both key and value now guarded. 2 more regression
  tests, 20/20 pass.
- **Pointer-listener cleanup fix was INEFFECTIVE** — the handlers weren't stable across renders, so
  the cleanup effect's empty-deps closure removed references that no longer matched what was
  actually attached to `window`; the leak persisted. Fixed properly with `useCallback` + a ref for
  the prop dependency, giving genuinely stable identity.
- Verified CORRECT: `AIExt_AppPages.sql` migration match, and the drag right/bottom-clearing fix
  (traced `JSON.stringify`'s undefined-dropping behavior all the way to persistence — no artifact
  risk).
- Verified INCOMPLETE (lower severity): the ID-casing rename only covered the 3 files it touched;
  plenty of pre-existing lowercase-Id instances remain elsewhere (not newly introduced this
  session, not fixed this round — a broader sweep is a separate task if wanted).

**New findings, this round:**
- **HIGH, fixed**: Task 7's typecheck actually FAILED when run for real (not just spot-checked
  file-by-file) — one pre-existing unused-parameter in `@doc-app/react` surfaced because the new
  cross-workspace alias pulls doc-app's whole source tree into app-studio's stricter
  `noUnusedLocals`. Fixed (renamed to `_knowledgeCollectionTypeID`, TS's own exemption convention).
  Also fixed: `package.json` was missing explicit `link:` entries for the new doc-app deps,
  inconsistent with every other cross-workspace dependency in that file.
- **MEDIUM, confirmed more serious than previously assessed, NOT YET FIXED**: `SeoAnalyticsTab.tsx`
  reads a widget's shared catalogue-template content instead of its real per-page override — the
  SEO checklist is "effectively non-functional" for real content (review's own words), not just a
  minor inconsistency.
- **MEDIUM, precisely traced, NOT YET FIXED**: canonical-URL divergence between `seoMeta.ts` (real
  render path) and `SeoAnalyticsTab.tsx` (reimplements the logic differently, always wrong for the
  default/home page).
- **MEDIUM/PLAUSIBLE, NOT YET FIXED**: Task 3 canvas-mode absolute positioning likely overlaps
  nested child sections' own rendered content (offsets resolve against the whole section's padding
  box, not "below the nested children").
- **LOW/PLAUSIBLE, NOT YET FIXED**: `assetEmbed.ts` has no URL-scheme allowlist on the PDF/link
  embed path (not currently reachable — asset URLs are server-minted, not user-typed — but no
  defense-in-depth guard either); `BackgroundTab.tsx` splices a raw URL into `url('...')` without
  escaping an embedded quote (currently caught only incidentally by the brace/at-rule guard).
- **LOW, NOT YET FIXED**: `reorderCanvasWidgetZIndex`'s two-call swap isn't atomic — a partial
  failure can leave both widgets on the same z-index instead of swapped.
- **LOW, REFUTED**: the port-collision-with-chatdesk concern from round 1 — checked the real port
  registry, 6111 is uniquely assigned, chatdesk has no assigned port anywhere (not yet built). Not
  a real, currently-reachable issue.

## FINAL CODE REVIEW COMPLETE, all findings fixed (2026-08-30)

Independent adversarial review of all 9 shipped tasks found 5 real issues, all now fixed by the
coordinator directly:

- **CRITICAL, fixed** — CSS injection via `injectResponsiveStyle` (`cssInjector.ts`): free-text
  `StyleProperties` fields (clipPath, backgroundImage, Task 3's position fields, etc.) were spliced
  into a real `<style>` tag served to real end users with zero sanitization, unlike the existing
  `css` escape hatch. Fixed with a new per-value `isUnsafeCssValue()` guard reusing the existing
  `UNSAFE_CSS_PATTERN`; an unsafe value is dropped, not the whole style. 4 new regression tests,
  all 18 tests in `cssInjector.test.ts` pass.
- **HIGH, fixed** — SSDT/live-DB drift recurred a 3rd time this session: `AIExt_AppPages.
  SEOConfiguration` existed live (migration applied) but was never added to the declarative
  `dbo\Tables\AIExt_AppPages.sql` source. Column + `CK_AIExt_AppPages_SEOConfiguration` constraint
  added to match.
- **MEDIUM, fixed** — drag-and-drop (Task 3) stretch bug: dragging a right/bottom-anchored widget
  left stale opposite-edge offsets in place, stretching the box. Drag commit now explicitly clears
  `right`/`bottom`.
- **MEDIUM, fixed** — leaked `window` pointermove/pointerup listeners on unmount mid-drag. Added
  proper cleanup effect.
- **LOW/MEDIUM, fixed** — CLAUDE.md "ID" casing violations in `useSectionActions.ts` and its two
  consumers (`draggedId`→`draggedID` etc.), across all 3 files.

Also independently reconfirmed clean (no fix needed): the undo/redo stack-clearing convention is
genuinely complete across every mutating action including Task 3's new ones; no git commits exist
anywhere in any of the 3 repos; the cross-workspace wiring (Task 7) has no circular deps and
correctly uses explicit paths.

Still open, not yet fixed (carried forward, lower priority than the above): SEO checklist
canonical-URL mismatch (Medium), port collision digital-assets-library vs chatdesk (Low),
heading-order edge case (Low), `SeoAnalyticsTab.tsx` reading the wrong widget-config source (Medium).

## ALL 9 TASKS (1-8 + the two follow-ups) NOW CODE-COMPLETE (2026-08-30)

Task 1 (verified end-to-end), Task 2, Task 4a, Task 4b, Task 5 (Critical bug fixed), Task 6, Task 7
(cross-workspace wiring now real, not just designed), Task 8, and Task 3 all shipped. Every package
touched typechecks clean. **Nothing has had a full live-browser E2E pass since Task 2's own
fetch-timeout fix** — Binoy's explicit direction was to build everything first and do ONE
consolidated verification + code review pass at the end, which is next.

Known open items going into that pass (not yet fixed, carried forward from earlier reviews):
- SEO checklist canonical-URL mismatch (Medium) — two different computations in two places.
- Port collision: digital-assets-library (6111) vs chatdesk (Low).
- Heading-order edge case in SEO checklist (Low).
- `SeoAnalyticsTab.tsx` reads content-widget HTML from the wrong config source (flagged by Task 4b's own agent, Medium-ish — a real widget's per-page override should be used, not the catalogue default).

## Wave 2/3 dispatched together (2026-08-30, per Binoy: "complete 6, 4b, 3, 7 and 8 without worrying about the blocks")

Parallel now: Task 6, Task 4b, Task 7, Task 8. Chained after Task 6 specifically (real technical
dependency, not file-collision caution — both Task 3's and Task 6's own design docs independently
concluded this): Task 3. Task 2's live-verification gate is dropped per Binoy's explicit direction —
Task 2's CODE is complete and no longer being actively edited by any agent, so the original
file-collision risk that justified waiting is gone; the live-verification gap is a confidence check,
not an active collision.

## Task 7 (new, queued 2026-08-30) — Media asset insertion in Content Widget + Style Builder

Not designed or built yet — queued as a combined follow-up, explicitly blocked on two prerequisites
landing first:
1. Digital-assets-library's public-asset backend actually working end-to-end (upload honoring
   `IsPublicAsset`, publish/unpublish, presigned-view for private) — fix in progress now.
2. The cross-workspace package consumption recipe from Task 1's design doc (doc-app packages reachable
   from app-studio, via the same relative-path-member + Vite-alias recipe already used for
   atlas-forms/bizfirst-common/expressions) actually being WIRED, not just documented.

Scope, once unblocked:
- **Content Widget**: add Insert Image / Insert PDF / Insert Video toolbar icons to the Tiptap inline
  editor (Task 2 Phase 2's landing-tab editor) — each opens a picker reusing digital-assets-library's
  own `AssetGrid` component, inserts an embed pointing at the asset's public URL. Only PUBLIC assets
  should be insertable this way (a private asset's URL would break for real site visitors) — the
  picker should filter to public-only, not just any asset.
- **Style Builder**: add an optional `onBrowseAssets?: () => Promise<string>` slot to the shared
  `StyleBuilderPanel` component (atlas-forms package) for `backgroundImage`-type fields — optional
  because `StyleBuilderPanel` is also reused by Atlas Forms consumers with no access to
  digital-assets-library; must not become a hard dependency for them.
- **Both pickers** (content widget + style builder): must also offer an "Upload new asset" link/button
  that opens digital-assets-library's own Upload UI in a new browser window/tab (not embedded inline)
  — added 2026-08-30 per Binoy's explicit request. This requires the Upload UI
  (`digital-assets-library`'s `UploadPage`/`AssetUploadPanel` composition) to actually be a properly
  extracted, reusable COMPONENT within the digital-assets-library app (not page-route-only logic) so
  the new-window link has a real, addressable URL to open — verify this is already true of the current
  `UploadPage.tsx`/`AssetUploadPanel.tsx` split (it looks like it already is, based on this session's
  earlier work, but confirm before building the picker against it).

Do not start building this until prerequisites 1-2 above are both confirmed landed.

## Explicitly OUT of this batch (flagged by the designs themselves, not silently dropped)

- **Task 4's analytics backend** (new `AIExt_AppPageViews` table, ingest endpoint, aggregation
  queries) — the design doc itself says this "should be sequenced as its own backend task, not
  bundled into the same PR as the SEO checklist UI." The app-wide tab will show an honest empty
  state ("Analytics requires backend work") until that separate effort happens.
- **Task 5 Phase 2** (making delete/reorder/rename/widget-config-save actions undo-safe by
  reworking them to defer persistence to Save) — the design doc calls this "genuinely the larger,
  riskier piece; needs its own design doc before starting." Only Phase 1 (move-section, section
  style edits) ships in this batch.

## Wave plan

**Wave 1 (dispatched 2026-08-30, running in parallel):**
- Task 1 build (already running, separate monorepo)
- Task 2 full implementation (~13-18 days, phased per its own doc — biggest item, dispatched first since everything else in `app-studio` waits on it)
- Task 5 Phase 1 (undo/redo, ~2-3 days)
- Task 4a (SEO fields + checklist + app-wide tab, no analytics backend)

**Wave 2 (dispatch once Task 2 lands):**
- Task 6 (responsive breakpoints, ~2.5-3.5 days)
- Task 4b (per-page SEO accordion, once Task 2's per-page panel exists)

**Wave 3 (dispatch once Task 6 lands):**
- Task 3 (free-form positioning, ~20-27 dev-days — the big one; Binoy has explicitly approved
  proceeding since it's additive/opt-in `layoutMode: 'canvas'` and doesn't touch existing flow-based
  rendering)

**Not scheduled (separate future effort, revisit only if Binoy asks):**
- Task 4c (analytics backend)
- Task 5 Phase 2 (Category B undo coverage)

## Usability mandate (applies to every wave)

Binoy: "dont forget, this is a usability project... I want matching usability [to Wix]." Every
design doc above was explicitly written against real Wix UX patterns (Task 2's overlay/floating
panel model, Task 3's flow-vs-canvas opt-in matching Wix Editor X's own evolution, Task 5's
session-local undo matching Wix's model, Task 6's 3-breakpoint device model matching Wix/Squarespace/
Webflow convention). When implementing, treat any deviation from the design doc's stated UX
behavior as a regression to flag back, not a shortcut to quietly take.

## Cross-cutting correction already applied

Digital-assets-library (Task 1) public asset URLs must NOT be keyed by the internal DocumentID/
AssetID — opaque non-sequential storage key + CDN-direct serving, DB touched once at upload time
only. Sent as a correction to the running Task 1 agent 2026-08-30. Verify this landed in the final
`01-digital-assets-library-design.md` before treating Task 1 as reviewed.

## Separate, later request (2026-08-30): AddWidgetModal code review — `workflow-template-category` / `chat-panel`

**Not part of the original six-task batch above.** Binoy separately asked (same day, later)
for a code review + fix of two `WidgetType` union members that had sat declared-but-unbuilt since
an earlier session: `workflow-template-category` and `chat-panel`, both flagged in
`AddWidgetModal.tsx`'s `NOT_YET_AVAILABLE` set. Outcome:

- **`workflow-template-category` — built and wired.** Genuinely buildable: its
  `WidgetRenderResult` variant, the `@bizfirst/ai-agent-catalog-ui` hooks
  (`useExecutionTemplatesByCategory`, `useExecutionTemplateCategories`), and the real backend
  (`BaseExecutionTemplateController`'s `by-category` action) all already existed and were already
  live/working — only the handler package + wiring was missing. New package
  `@app-studio/widget-workflow-template-category`
  (`app-studio/packages/widget-handlers-workflow-template-category-widget`), registered in both
  `useLivePreviewEngine.ts` and `app-player/src/App.tsx`, removed from `NOT_YET_AVAILABLE`, real
  config sub-form added to `AddWidgetModal.tsx`. Live-verified against a real running app (AppID
  1526, Qoboto) with a real seeded `ExecutionTemplateID` in category 1 ("Chat Agent") — rendered
  correctly, three real API calls all `200`, scratch data cleaned up afterward. See that package's
  own `DevelopmentHistoryLog.md`.
- **`chat-panel` — confirmed still genuinely blocked, correctly left unbuilt.** Not a scope
  decision — a real prerequisite gap. `WidgetRecord.ts`'s design intent depends on an extracted
  `@bizfirst/chatdesk-chat-window` package that has never been built; `ChatWindow`/
  `ChatWindowContainer` exist only as app-local components inside
  `src/chatbots/apps/chatdesk/src/components/`, and even that app's own conversation-resume flow
  has an open architectural gap (no server-side `ConversationID`→`EngageSessionID` lookup). Left
  correctly flagged in `NOT_YET_AVAILABLE`; `dev-userguide/widget-types/chat-panel.html` updated
  with the confirmed (not just suspected) blocker trail.

**Collision note**: this pass ran concurrently with the Task 9 Toolbox-panel /
cross-section-drag-and-drop agent, which also touched `WIDGET_TYPE_REGISTRY.ts` and
`AddWidgetModal.tsx` mid-session (extracted widget-creation logic into a new
`widgets/addWidgetActions.ts` shared module). No clobbering occurred — the Toolbox agent's
extraction carried this pass's `workflow-template-category` additions through verbatim; this pass
completed the resulting `AddWidgetModal.tsx` wiring once the extraction landed. As of this note,
`app-studio-designer-components-react`/`app-studio-designer` still fail `typecheck` for reasons
entirely inside the Toolbox agent's own still-in-progress files (`canvas/SectionCard.tsx`,
`canvas/WidgetChip.tsx`, `details/SectionDetailsContent.tsx`) — unrelated to this pass's own
changes, which typecheck clean on their own and in every other consuming package (26/28 packages
clean workspace-wide).

## FOURTH independent code review COMPLETE + all findings resolved, workspace-wide re-verified (2026-08-30)

Ran after the widget-toolbox (Task 9), widget-type-registration, Canvas-layout-mode-fix, and
`InsertFromFile`-test-fix agents had all landed. Explicitly re-verified rounds 1-3's fixes rather
than trusting them (the same discipline that's now caught something real in all four rounds):

- **CRITICAL, FIXED**: round 3's hex-escape decode-then-validate fix in `cssInjector.ts` was
  ITSELF bypassable via a chained escape — `\5c` decodes to a literal backslash, which combined
  with adjacent plain hex-digit text reconstructs a brand-new `\XX` escape a single decode pass
  never re-scans. Proven with a passing exploit. Fixed: decode to a fixpoint (loop until stable),
  not once. 2 new regression tests (27/27 total pass in that file).
- **Re-verified STILL HOLDS**: round 3's nested-child-section canvas fix, the `BreakpointService`
  wiring (confirmed `injectResponsiveStyle`'s `@container` path genuinely doesn't need it —
  `LivePreviewPanel`'s device-width container makes both mechanisms agree independently), the
  `LocalDocumentStorageProvider` sibling-directory fix.
- **HIGH, FIXED**: canvas-mode sections had NO way to acquire height and collapsed to ~0px outside
  studio mode — their default state immediately after toggling to Canvas, not an edge case. New
  `AppSection.canvasHeight` field, applied as the widget wrapper's `minHeight`; Designer's Layout
  Mode panel seeds a sensible 400px default the moment a section switches to Canvas and exposes a
  field to adjust it. 2 new regression tests.
- **MEDIUM, FLAGGED, RESOLVED 2026-08-30 (later same day)**: `reorderCanvasWidgetZIndex`'s
  atomicity fix (and the identical pattern in `moveAppWidget`/`reorderWidgetsInSection`/
  `removeSection`/`renameSection`) assumes a rejected API promise means the write never committed —
  traced the real backend and found that's not guaranteed (a post-write re-fetch/response
  round-trip can time out AFTER the write already committed, a documented real condition on this
  dev machine). Intermittent, needs a real backend stall to manifest. See the "MEDIUM finding
  RESOLVED" writeup below for the fix and verification evidence. Full original writeup in
  `app-studio-store-react/DevelopmentHistoryLog.md`.
- **LOW, FIXED (2 of 2)**: `GetSafePath`'s doc comment overstated that `Path.GetFullPath` resolves
  symlinks (it doesn't — purely lexical, not currently exploitable but the claim was factually
  wrong); `structuralGeneration`'s doc comment claimed undo/redo/rename-section/move-section bump
  it directly (they don't — currently harmless only because those actions also change `layout`'s
  own reference, which the reload effect separately depends on; comment corrected with an explicit
  warning against removing that redundant dependency).
- **LOW, FIXED**: `widgetNames` (widgetID -> name) never refreshed on rename via the new light-save
  path — Section list/Section Details/App Config properties all showed stale names until a full
  reload. `applyWidgetSaveLocally` now keeps it in sync.

**Also fixed this same pass, user-reported, unrelated to the review**: semi-transparent Section/
Widget Details popups (`OverlayPanel.tsx` — the same CSS-variable bug already fixed once in
`Drawer.tsx` had resurfaced there); a new View menu "Enable Preview Panel" toggle (off by default)
replacing the always-mounted, always-empty `StateInspectorPanel`; the backend WebApi process
restarted twice today under real memory pressure (same root cause both times — this dev box is
genuinely ~7.7GB RAM and gets squeezed hard by concurrent agents' browser automation).

**Workspace-wide re-verification, done AFTER all of the above landed together** (not just each fix
in isolation): `pnpm -r typecheck` from `src/app-studio` — **all 54 workspace projects pass, zero
errors** (this supersedes the "still fails" note in the workflow-template-category section above,
which was written mid-flight before the toolbox agent's own work finished). `vitest run` in
`app-handlers-generic` — 50/50 tests pass. `dotnet test` in `BizFirstFi.Go.Documents.Tests` —
142/142 pass.

**Status: every review-4 finding is now either fixed or explicitly, deliberately flagged as an
open design decision (the MEDIUM item above) — nothing was silently dropped.** Given four
consecutive rounds have each found something real the last one missed, a fifth pass is worth
considering, but genuinely not yet requested — the next actual step is the live E2E verification
pass Binoy asked for ("fix all issues and test e2e"), which still hasn't happened this entire
batch despite everything above being fix/typecheck/unit-test verified.

## MEDIUM finding RESOLVED (2026-08-30, later same day) — "rejected promise" doesn't guarantee "write didn't happen"

Fixed the item flagged MEDIUM in the "FOURTH independent code review" section above. Root cause:
`BaseAppStudioAppWidgetController.Update` (`BizFirstPayrollV3`) did a second, response-body-only
`GetByIdAsync` after the real write — that extra round trip, not the write itself, was what could
stall/time out and flip an already-committed write into a client-side rejection.

**Fix**: removed the second GET; `Update` now returns the controller's own already-mutated
in-memory `AppWidget` directly. Safe because that object is the SAME reference the framework's
`SetAuditFieldsOnUpdate` mutates before save (verified by tracing `ConvertToEntity`/
`SetAuditFieldsOnUpdate` in the shared `BaseRepository`), and `AppWidget` has no DB-computed
columns a re-fetch could add. Also verified none of the five affected
`app-selection.store.ts` actions (`moveAppWidget`/`reorderWidgetsInSection`/
`reorderCanvasWidgetZIndex`/`renameSection` via `Update`; `removeSection` via `Delete`, which
never had this defect to begin with) read the response body at all — they only check
fulfilled/rejected — so nothing on the frontend needed to change.

**Verified**: backend `dotnet build -m:2 -nodeReuse:false` clean (0 errors) on both
`BizFirst.Ai.AppStudio.Api.Base.csproj` and `BizFirst.Ai.Platform.Web.Server.Core.csproj` (the
project that hosts the concrete controller the running Consolidated WebApi actually serves).
Frontend `pnpm -r typecheck` — all 54 workspace projects clean. **Not live-E2E-verified against
the running app** (e.g. qoboto/AppID 1526) — the Consolidated WebApi process on this box runs a
pre-built DLL rather than `dotnet watch`, so exercising this live would require restarting shared
infrastructure other concurrently-active review agents in this session (`review-db-migrations`,
`review-digital-assets-app`, `review-seo-feature`, `review-undo-redo-feature`) may depend on —
judged unsafe to do unilaterally. Full detail in `app-studio-store-react/DevelopmentHistoryLog.md`
and `BizFirst.Ai.AppStudio.Api.Base/DevelopmentHistoryLog.md` (both same date).

## Designer canvas vs preview background-color bug — investigated and FIXED (2026-08-30/31)

New, separate bug report from Binoy, not part of the wix-style-design six-task batch or any of the
four review rounds above: "I see a serious issue with the designer vs preview app. On the preview
app, many sections have bg color white. however on the designer i dont see that. Are you actually
applying the app style, section style and widget styles to the designer? Also the app style must
only be applied inside the designer canvas and not for the whole app studio." Two distinct claims,
investigated independently with live computed-style evidence before any code change (Qoboto,
AppID 1526, Designer `:6109` vs the real `app-player` runtime `:6130`).

**Claim 1 (Designer canvas not showing real section/widget backgrounds) — CONFIRMED, real bug,
FIXED.** Pulled `getComputedStyle(...).backgroundColor` on the same `[data-section]` elements in
both hosts. Before the fix, every section in the Designer canvas computed the identical
`rgba(255, 255, 255, 0.024)` regardless of its real color (`hero` real `rgb(10,14,26)`,
`showcase-search`/`recent-card-*`/`ideas-that-matter`/`product-grid` real `rgb(255,255,255)`,
`header` real `rgb(245,246,248)` — all in the real app-player, all correct there). Root cause:
`AppPlayer.tsx`'s `AppSectionRenderer` studio-mode-only `borderStyle` unconditionally set
`background: 'rgba(255,255,255,0.025)'` (meant only as a placeholder tint for empty/sparse
sections); `StyledSlot` merges the caller's `style` prop on top of the section's real style by
design, so this placeholder silently clobbered every section's real, author-configured background
the instant `studioMode` was on. Both hosts render through the exact same component/data — this
was never a "styles aren't reaching the Designer" gap, it was the Designer's own studio chrome
overwriting them after the fact. **Fix**: the placeholder now only applies when the section has no
real background of its own (new `styleSlotHasBackground()` helper). **Live-verified after the
fix**: Designer canvas now computes byte-identical values to the real app-player for every section
checked, and genuinely backgroundless sub-sections still correctly get the placeholder tint
(regression-checked). Full writeup, code, and before/after evidence table in
`app-handlers-generic/DevelopmentHistoryLog.md` (2026-08-30 entry).

**Claim 2 (app-level style leaking outside the Designer's own canvas into its chrome) —
INVESTIGATED, REFUTED as currently reproducible, nothing to fix.** Traced `SiteStyleConfig`
(`App.styleConfiguration`, `siteContainer`/`siteBackground`) end-to-end: it's edited in
`AppStyleEditor.tsx` (App Config's "Style" tab) and persisted, but has zero render consumers
anywhere in the codebase — confirmed by grep across all of `app-studio`, still matches
`StyleSlot.ts`'s own long-standing doc comment that this was deliberately left unwired. It cannot
leak into Designer chrome because nothing renders it at all yet, canvas or otherwise. Also
confirmed `cssInjector.ts`'s class-scoped CSS injection is properly scoped and can't leak outside
elements carrying its generated class. No code change made for this claim — reported honestly as
not currently reproducible rather than building a fix for a non-issue. Wiring `SiteStyleConfig`
into the real render path, scoped strictly to the canvas region (not the whole Designer chrome),
remains a genuine, still-open gap for whenever that feature is actually built — worth remembering
as a requirement at that time, not now.

**Verification**: `pnpm -r typecheck` clean across all 54 workspace projects. No git commits (per
CLAUDE.md's git policy — not requested this session).

## If resuming this session cold

1. Check which design docs (01-06) exist and whether `01-...md` reflects the asset-URL correction.
2. Check this file's "Build status" column against actual code state (`git status`/`git log` in
   `app-studio` and `doc-app`) — an agent may have completed a wave since this file was last updated.
3. Do NOT re-dispatch a wave whose predecessor hasn't actually landed in code yet — re-verify via
   the codebase, not just this file's staleness.
4. No git commits have been authorized for any of this work — everything should be sitting as
   uncommitted changes for Binoy's review, per CLAUDE.md's git policy.
