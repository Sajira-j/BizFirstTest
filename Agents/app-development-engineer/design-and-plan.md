# App Studio Automation — Design & Plan

Status: living doc, written 2026-08-30 covering a session that ran from "clone qoboto.com as a real
App Studio app" through an 8-task "wix-style-design" usability series. `architecture.md` in this same
folder is the reference this plan builds against; `lessons/README.md` captures concrete findings as
each piece landed; `STATUS.md` is the current live state.

## Phase 0 — The qoboto clone (foundation for everything after)

Binoy's ask, in the order it actually arrived: build a website-builder replica of qoboto.com as a
real App Studio app (initially considered original content, then corrected — "qoboto is owned by us
- so clone as is," verbatim content/copy authorized, real images/logo) — with one explicit
architectural constraint: **style it using App Studio's real structured Style Builder property
system, not raw hand-written CSS**, specifically so the result is a clean, reusable template (via
App Studio's own "Clone App" feature) that "anyone can clone from this and create their own web
site," not a one-off page.

**Delivered**: AppID 1526, AppCode `qoboto`, linked to a real `Project_Projects` row (ProjectID 1058,
"Qoboto" — Binoy caught that an App had no required Project and asked for the data fixed, explicitly
declining a schema constraint change) — a genuinely nested `AppSection.appSections` tree (header →
header-brand/header-nav, hero → hero-left/hero-right, recent-websites → recent-websites-grid → 4
recent-card-* children, etc.), fully Style-Builder-authored (no `css` field usage in the real
Configuration JSON beyond what the structured `style` object naturally produces).

Building this surfaced the nested-section Designer gap documented in `architecture.md` §2 — the
single most consequential bug found this session, since it silently made a real, already-shipped
data-model feature invisible in the tool meant to edit it.

## Phase 1 — Wix-usability design series ("wix-style-design", 6 initial tasks + 2 added mid-flight)

Binoy, after reviewing the qoboto clone against real Wix and against the pre-existing Designer UI:
"The current setup less than usable for users compared to Wix... I need the best user experience -
otherwise nobody will use apps like this." Directed six parallel research-and-design passes (each its
own numbered doc under `Documentation\Employees\agentic-coding\docs\wix-style-design\`), reviewed
together in one consolidated pass before any implementation started (explicit instruction: avoid
shared-file collisions between parallel implementation agents), then built in dependency order.

| # | Task | Design | Build status (see `STATUS.md` for the live detail) |
|---|---|---|---|
| 1 | Digital Assets Library (Wix-Media-Manager equivalent) | ✅ | ✅ Complete, frontend + backend, public-serving verified end-to-end |
| 2 | Click-to-edit unified Designer (no mode-switching) | ✅ (~13-18 eng-days) | ✅ Complete, all 6 phases |
| 3 | Free-form drag-and-drop positioning | ✅ analysis (~20-27 dev-days) | Not started — sequenced last (see below) |
| 4 | SEO checklist + analytics panel | ✅ | 4a (fields/checklist/app-wide tab) complete; 4b (per-page accordion) blocked on Task 2 finishing (now unblocked); 4c (analytics backend) explicitly out of scope |
| 5 | Undo/redo | ✅ | Phase 1 (move-section, section-style edits) complete + a Critical corruption bug found and fixed post-ship; Phase 2 (full Category-B coverage) explicitly deferred |
| 6 | Responsive breakpoint editing | ✅ (~2.5-3.5 days) | Not started |
| 7 | Media-asset insertion (Content Widget + Style Builder), added 2026-08-30 | Scope fully specified in `wix-style-design/00-implementation-status.md`, no separate doc | Not started — blocked on Task 1's backend (done) + the cross-workspace package wiring (`architecture.md` §6, not yet wired) |
| 8 | "View" menu → Always Show Labels toggle, added 2026-08-30 | Scope fully specified, no separate doc needed | Not started |

**Sequencing logic, not arbitrary** (full reasoning in `wix-style-design/00-implementation-status.md`):
Task 2 relocates the panel Task 3's own design says its Flow/Canvas toggle needs to live in → **2
before 3**. Task 3's own doc AND Task 6's own doc *independently* concluded the same thing — agree the
`{desktop, tablet?, mobile?}` coordinate schema before either ships → **6 before 3**. Both converge on
**2 → 6 → 3**, not designed that way on purpose (six independent agents wrote these docs) — a good
sign it's the real dependency graph, not a guess. Task 1 lives in a wholly separate monorepo
(`doc-app`) — zero file overlap, ran fully in parallel throughout. Task 5's small Phase 1 ran in
parallel with Task 2's start, expected (correctly) to finish before Task 2's own toolbar edits
landed.

**The "usability mandate" governing every task**: "dont forget, this is a usability project... I
want matching usability [to Wix]." Every design doc was explicitly checked against real Wix UX
patterns (Task 2's floating-overlay model, Task 3's flow-vs-canvas opt-in matching Wix Editor X's own
retreat from pure free-position, Task 5's session-local undo matching Wix's own model, Task 6's
3-breakpoint device model matching Wix/Squarespace/Webflow convention) — a deviation from a design
doc's stated UX behavior is meant to be flagged back as a regression during build, not quietly taken
as a shortcut.

## Phase 2 — Backend correctness pass (parallel, cross-cutting)

Not a numbered wix-style-design task, but real, necessary work discovered while building Task 1:

- **Public-asset URL security correction** (mid-flight, before any code shipped): public asset URLs
  must never be keyed by the internal `DocumentID` — opaque, non-enumerable key only, database
  touched once at publish time, never per view. Sent as an explicit correction to the build agent
  before it finalized its design; verified landed.
- **Full digital-assets-library backend**: upload endpoint honoring `IsPublicAsset`, publish/
  unpublish endpoints, presigned-view for private assets, and the actual byte-serving public route —
  the last of these hit a real routing bug (see `lessons/README.md`) before being fully verified live
  (a real anonymous `curl` against a published test asset: `HTTP 200`, correct bytes, correct
  content-type, zero auth header, opaque key only in the URL).
- **Three rounds of schema drift** between the live dev database and the declarative SSDT source of
  truth, found and fixed (see `lessons/README.md` for the concrete mechanism each time and why it
  matters to check both directions).
- **Independent adversarial code review** (a fresh agent, no shared context with whoever built the
  feature) of everything shipped so far — found one Critical bug (undo/redo corruption) and one
  Medium bug (a canonical-URL computed two different, disagreeing ways in two places) — both real,
  neither hypothetical.

## What's explicitly NOT in scope, per Binoy's own direction (don't build blind)

- Task 4c (analytics backend — new pageview table, ingest endpoint, aggregation queries) — its own
  design doc says this should be its own separate backend effort, not bundled with the SEO checklist
  UI.
- Task 5 Phase 2 (making delete/reorder/rename/widget-config-save undo-safe) — its own design doc
  calls this "genuinely the larger, riskier piece; needs its own design doc before starting."
- Anything touching `AppPlayer.tsx`'s shared rendering package beyond additive, opt-in fields — this
  package is also production `app-player`'s real runtime; every task this session that touched
  layout/rendering behavior deliberately scoped itself to stay additive for exactly this reason
  (confirmed working: qoboto, built entirely pre-nested-section-fix, still renders byte-identical
  after every subsequent change).

## Process lessons baked into how this plan is executed (see `lessons/README.md` for the concrete evidence)

- **Parallel background agents editing adjacent files is a real collision risk, not theoretical** —
  sequence by actually-traced file overlap, not just "run everything in parallel."
- **Every agent doing live browser verification needs its OWN tab** when more than one agent may be
  testing concurrently — sharing one tab produces false-negative verification (one agent's UI state
  contaminating another's).
- **This dev box is genuinely memory-constrained (~7.7GB total)** — expect intermittent backend
  stalls under concurrent build/agent load as a standing environmental fact, not a code smell to chase
  every time it happens; but also verify it really is environmental (a live route-table dump, a real
  curl test) before writing it off, since a genuine code bug can hide behind the same symptom.
