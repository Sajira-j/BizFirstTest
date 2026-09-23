# App Studio Automation — Status

Living status file. Updated as work completes — single source of truth for what's done vs. not, not
a changelog. See `design-and-plan.md` for the plan and `lessons/README.md` for accumulated findings.
`architecture.md` is the end-to-end reference. The authoritative, most granular live-status file for
the wix-style-design build phase specifically remains
`Documentation\Employees\agentic-coding\docs\wix-style-design\00-implementation-status.md` — this
file summarizes it, doesn't replace it.

## Done

- **Qoboto clone (Phase 0)**: AppID 1526, AppCode `qoboto`, linked to a real Project record
  (ProjectID 1058), styled entirely through the structured Style Builder (no raw-CSS shortcut used),
  built as a reusable Clone-App template per Binoy's explicit "anyone can clone from this" goal.
  Uncovered the nested-`AppSection.appSections` Designer gap (see `architecture.md` §2) — fixed via
  the shared `sectionTree.ts` helper module, now the single place all section-tree logic lives.
- **All 6 wix-style-design design docs** (Tasks 1-6) — written, consolidated-reviewed together for
  file-overlap risk before any implementation started, sequencing derived from that review (2 → 6 →
  3, with 1/4/5 running in parallel where genuinely independent).
- **Task 2 (click-to-edit unified Designer) — all 6 phases built and typecheck-clean.** Unified the
  old 4-way body switch (app-picker / App Config / Layout Design / Pages) into 2 (app-picker / unified
  canvas), retired `PageCanvas.tsx`, added an `@floating-ui/react` overlay replacing the docked
  Details panel, inline Tiptap content editing with save-on-dismiss, Pages + Site Structure drawers, an
  in-canvas Add Section affordance, and page-level URL routing (`/{appCode}/page/{slug}`). Four rounds
  of live UI feedback (fullscreen toggle on all modals/drawers, a Site Structure transparency bug, an
  `AddWidgetModal` stacking-context bug, a follow-up z-index bug) all fixed with a proper shared
  `zIndexScale.ts` rather than ad hoc numbers.
  - **A real regression was found and fixed post-"complete"**: a fresh-tab load of qoboto got
    permanently stuck on a loading state with no error, root-caused to `fetchJson()` having zero
    client-side timeout (see `lessons/README.md`) — fixed with `AbortSignal.timeout(30_000)`.
  - **Live-verified for real, end to end, after the fix**: navigated a genuinely fresh tab to
    `/qoboto/page/home` with a real (not fake) logged-in session; all underlying API calls resolved
    200; the toolbar correctly showed "Qoboto - Decentralized Website Builder"; the canvas rendered
    the real layout. Confirmed working.
- **Task 5 Phase 1 (undo/redo)** — `undoStack`/`redoStack` via immer patches, `moveSection` +
  section-style edits covered, toolbar buttons + Ctrl+Z/Shift+Z/Y. **A real Critical corruption bug**
  was found by an independent code review after initial "complete" and fixed the same day (see
  `lessons/README.md`) — every untracked mutating action now clears the undo/redo stacks to prevent a
  stale patch from replaying against structurally-changed state.
- **Task 4a (SEO checklist + fields + app-wide tab)** — `AppPageSeo` type, 10-row computable
  checklist, title/meta-tag wiring into `app-player` (a real, previously-missing capability — live
  confirmed via the browser tab title updating correctly), honest analytics empty-state (no fake
  data). One Medium bug found by the code review (canonical URL computed two different, disagreeing
  ways in two places) — not yet fixed.
- **Task 1 (Digital Assets Library) — fully complete, frontend AND backend, live-verified end to
  end.** New app (`digital-assets-library`, port 6111), ~70% reuse of existing `document-manager`
  components, upload UI with a Public/Private visibility choice (Type field auto-locked to the
  DIGITAL_ASSET document type, matching the app's single-purpose design). Backend: upload now honors
  `IsPublicAsset`, publish/unpublish endpoints, private presigned-view, and the actual anonymous
  byte-serving public route — the last of these hit a real missing-controller-copy bug (see
  `lessons/README.md`), fixed, and **live-verified with a raw `curl`**: `HTTP 200`, 56 real bytes,
  correct content-type, zero auth header, only the opaque key in the URL (no `DocumentID`). The public
  asset URL security correction (opaque key, never the internal ID) was applied and verified before
  any of this shipped.
- **Real-time UI feedback loop with Binoy throughout**, all addressed: fullscreen-toggle requests,
  a z-index stacking bug, "type is always Digital Assets" (pre-fill + disable the upload Type field),
  and the "should we support click-to-edit media insertion in the content widget / style builder"
  design discussion (resulted in Task 7, scoped and queued).
- **Independent adversarial code review** (fresh agent, no shared context) of everything shipped so
  far — 2 Critical, 2 Medium, 5 Low findings. Both Critical findings (undo/redo corruption, and the
  SSDT-vs-live-DB drift on `Doc_Documents`' new public-asset columns) fixed. `widgetId`→`widgetID`
  casing and a hardcoded document-type-code string (should use the shared constant) also fixed. Still
  open: the canonical-URL mismatch (Medium), a port collision with an unrelated app (`digital-assets-
  library` and `chatdesk` both default to port 6111 — Low, not yet reconciled), a heading-order
  edge-case in the SEO checklist (Low).
- **Real infrastructure cleanup**: killed 11 leftover MSBuild build-server nodes eating ~350MB on this
  memory-constrained (~7.7GB) dev box, applied 3 rounds of DB migrations to close schema drift between
  the live dev DB and the declarative SSDT source (in both directions — see `lessons/README.md`).
- **A Wix vs. Vercel Hobby vs. App Studio comparison chart**, built as a real HTML file (not an
  externally-hosted Artifact, per project policy) at
  `Documentation\Employees\agentic-coding\docs\wix-style-design\comparison-wix-vercel-appstudio.html`
  — App Studio's column filled honestly from actually-verified capabilities, not aspirational claims.

## Not started

- **Task 3 (free-form drag-and-drop positioning)** — design complete (~20-27 dev-days estimated),
  approved to build (additive/opt-in, Binoy explicitly not concerned about the size given it doesn't
  touch existing flow-based rendering), sequenced last (after Task 6) per the shared-file-overlap
  analysis. Not yet dispatched.
- **Task 6 (responsive breakpoint editing)** — design complete (~2.5-3.5 days estimated, smaller than
  originally guessed), not yet dispatched. Now unblocked (Task 2 is done and live-verified).
- **Task 4b (per-page SEO accordion)** — blocked on Task 2's per-page details panel, which now
  exists. Not yet dispatched.
- **Task 7 (media-asset insertion in Content Widget + Style Builder)** — scope fully specified,
  blocked on the cross-workspace package-consumption wiring (`architecture.md` §6) being actually set
  up, not just documented. Digital-assets-library's backend prerequisite is now done.
- **Task 8 ("View" menu → Always Show Labels toggle)** — scope fully specified (one toggle, not two;
  default hover-only reveal for both labels and the action button, matching Wix/Webflow/Squarespace/
  Framer convention), small enough to build directly, not yet dispatched.
- **Task 4c (analytics backend)** and **Task 5 Phase 2 (full Category-B undo coverage)** —
  deliberately out of scope for this batch per each task's own design doc; revisit only if asked.

## Real, currently-known open items (not blockers, just not yet closed)

- The canonical-URL Medium bug (Task 4a) and the port-collision Low finding (digital-assets-library
  vs. chatdesk, both defaulting to 6111) from the code review — not yet fixed.
- Which concrete service should permanently back `IDocumentStorageProvider`/public-read ACLs
  (`architecture.md` §8) — the interim `LocalDocumentStorageProvider` works and is live-verified, but
  is explicitly flagged as swappable, not a final answer.

## Environment notes for whoever picks this up next

- This dev box has ~7.7GB total RAM and runs hot under concurrent agent/build load — expect
  intermittent backend stalls as a standing fact, not automatically a code bug (but verify each time,
  don't just assume — see `lessons/README.md`'s first two entries for how to tell the difference).
- No git commits exist for any of this work, in either `app-studio`/`doc-app` (BizFirstAiStudio) or
  the DB migration files (BizFirstFiV3DB) — everything is sitting as uncommitted changes for Binoy's
  review, per this project's standing no-auto-commit policy.
- DB migrations this session live under `BizFirstFiDB\BizFirstFiV3DB\BizFirstFiV3DB\dbo\Data\` (NOT
  `Migrations\` — Binoy's explicit preference this session, a deliberate deviation from the
  pre-existing convention that folder name embodies) for the `doc-app`-side fixes, and under
  `BizFirstFiDB\AppStudio\migrations\` (a separate, pre-existing convention) for the app-studio-side
  SEO column migration — two different migration-folder conventions coexist in this codebase, know
  which one a given feature actually uses before writing a new migration.
