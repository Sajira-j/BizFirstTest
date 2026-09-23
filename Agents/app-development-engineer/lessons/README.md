# Lessons — App Studio Automation Project

Running log of concrete learnings from actually doing this work, not a restatement of the design.
Add an entry per task/finding as it lands. Newest first. Bias toward the surprising and the concrete
— this is for the next person/agent picking up similar work.

## 2026-09-10/11 — Why App Studio sites end up looking unstyled: the real design-control path exists but isn't discoverable

Directly diagnosed after a client-facing demo site (a 4-page biography app) was rated 1/10 for
professional appearance by an external reviewer. The design/layout WAS achievable in App Studio —
the site just never used the mechanism that makes it possible.

**The Content Widget's default HTML sanitizer (`ContentSanitizer.ts`, DOMPurify-backed) is
extremely restrictive**: no `style` attribute, no `<style>` tag, and only a small tag/attr allowlist
(`p, br, strong, em, u, s, a, ul, ol, li, h1-h6, blockquote, pre, code, hr, img, table*, div, span` /
`href, src, alt, title, class, id, target, rel`). `class`/`id` are useless without an external
stylesheet hook, which doesn't exist for this widget. Any HTML authored this way renders in raw
browser-default styling — this alone explains "wall of plain text, no hover states, no real layout."

**The fix is a real, already-shipped feature, not a hack**: `ContentWidgetConfig.allowScripts: true`
switches DOMPurify to its full default profile — real `<style>` tags, inline `style`, everything.
This is the ONLY way to get hover states, CSS Grid/Flexbox layouts, gradients, or custom typography
in a content block. **Nothing in the Designer UI surfaces this as "the way to get real design
control"** — it reads as a scripting/security toggle, not a styling one, so a non-technical author
building a site by hand would never find it. A benchmark rebuild (cloning the structure of a
professional marketing-site template) using `allowScripts: true` on every content widget, each with
its own scoped `<style>` block (see next entry for the scoping rule), produced a dramatically more
polished result than anything the restricted mode could reach.

Separately, App Studio DOES have a real *structured* styling system independent of content widgets —
`AppSection.sectionContainer`/`sectionBackground` and `AppWidget.styleConfiguration.widgetContainer`
(Level 2/3 "style slots," each `{style?: object, css?: string}`) — but `css` here is NOT a real
stylesheet: it's a single declarations-block with no braces/selectors/@-rules allowed at all
(`sanitizeScopedCss`/`UNSAFE_CSS_PATTERN` in `cssInjector.ts` reject anything containing `{`, `}`,
`@`, even CSS hex-escaped, after three escalating security-review rounds finding bypasses). The
`style` object (real `StyleProperties`, ~90 keys) is far more capable — it accepts free-text values
for `backgroundImage` (gradients), `boxShadow`, `transform`, `transition`, `filter`, `fontFamily` —
but it still only ever styles ONE element (no pseudo-classes, no nested selectors, no `@keyframes`).
`AppSection.widgetLayout` (`{direction, wrap, gap, align, justify}`) is the correct mechanism for
arranging MULTIPLE WIDGETS in one section as a flex row (a 3-card grid, a team-photo row) — a
sibling but separate field from `sectionContainer`; using `sectionContainer.style.display:'flex'`
for this instead silently shrink-wraps the widgets to their own content with dead space beside them
(see the 2026-09-04 fix in `AppPlayer.tsx`'s `SECTION_CHILD_FILL_STYLE` — `sectionContainer` was
never meant to reach the widgets *inside* the section, only style the section's own box).

**Lesson: when asked "why can't we build a professional-looking site in App Studio," the honest
answer is a UI-discoverability gap, not a platform-capability gap.** The real fix is either (a)
surface `allowScripts` in the Designer's widget-editor UI as an explicit "Advanced: full custom
HTML/CSS" toggle with a short explanation, or (b) build a library of pre-styled, ready-to-drop
content-widget templates (hero, card-grid, portfolio-grid, timeline, team-grid) that already use
`allowScripts: true` internally, so an author never has to discover the flag themselves.

## 2026-09-10/11 — Content Widget's true configuration contract drifted into THREE incompatible shapes across the same system, silently rendering empty with zero error

`ContentWidgetHandler.render()` (`widget-handlers-content-widget/src/ContentWidgetHandler.ts`) only
ever reads `config.content` (string) and `config.format` ('html'|'markdown'|'text') — confirmed by
reading the handler directly. `BaseWidgetLoadHandler.resolveConfig()` does a bare
`{...defaults, ...raw}` shallow merge with NO legacy-key migration of any kind. Found three
different shapes actually stored in the same production DB for the identical widget type:
1. The MCP `create_widget` tool's own documented example: `{content, format}` — the ONLY shape the
   handler actually reads.
2. A very old widget (WidgetID 1): `{contentType, content}` — happened to still render correctly by
   accident, only because its `content` key was spelled right and its format defaulted to 'html',
   which matched the actual content type.
3. **Every widget an earlier session built for a real, live demo app**: `{markdown: "..."}` — no
   `content` or `format` key at all. This rendered EMPTY, silently, with no console error, on every
   page of a real client-facing site (confirmed by direct screenshot — text and hero image both
   fully invisible) until traced back to this exact mismatch and corrected in the DB.
4. The Designer's own drag-to-add default (`addWidgetActions.ts`'s `buildNewWidgetConfiguration`)
   ALSO wrote the wrong `{contentType, content}` shape until fixed — meaning even the "proper" UI
   path was producing widgets with a dead `format` field, latent until someone typed non-HTML
   content into one.

**Lesson: a config-shape mismatch for this widget type fails completely silently — empty render,
no thrown error, no console warning — because `resolveConfig`'s shallow merge treats an unknown key
as harmless extra data rather than a shape violation.** Before trusting ANY existing widget's
Configuration JSON as a template to copy, verify it against the handler's actual `DEFAULTS` object
and read path, not just against another widget that happens to render — a widget that "looks like
it works" (e.g. defaults to 'html' format and happens to contain a valid HTML fragment as an
unrelated stray key) can still be silently wrong for markdown/text content.

## 2026-09-10/11 — `isPrimaryContentSection` is per-APP, not per-section-name, and getting it wrong makes real content invisible with no error on either side

Confirmed via live testing plus a full read of `RouteResolverService.ts`/`AppPlayer.tsx`. Only ONE
section per app can be flagged `isPrimaryContentSection: true` (`findContentPaneSection()` does an
early-return on the FIRST match in `layout.appSections`). `AppWidgetRecord.appPageID` is ONLY
meaningful for widgets inside that one section — a widget with a real `appPageID` sitting in any
OTHER section is invisible everywhere: `AppPlayer.tsx`'s `getWidgetsForSection()` shows only
`appPageID == null` widgets for every non-primary section, unconditionally, regardless of which
page is active. Two real, independent bugs both trace to this exact model:

1. **Data-side**: a real app's actual page content (Biography/Legacy/Discography text, a hero
   image) was built inside sections named `main`/`hero` — reasonable-sounding names — while the
   app's own `layout.appSections` config had flagged a *differently-named* section
   (`content-primary`) as the real primary. Every one of those widgets had a correct, real
   `appPageID` set — and was still invisible on every page, because it lived in the wrong section.
   Always read `AIExt_Apps.Configuration`'s `layout.appSections` FIRST to find which section name
   is actually flagged `isPrimaryContentSection: true` for a given app — never assume based on a
   section's name (`main`/`main-content`/`content-primary` have all been used for this role across
   different apps built at different times).
2. **UI-side, in the Designer's own canvas**: dropping a widget onto the live canvas's primary
   section never stamped the CURRENTLY OPEN page's ID onto the new placement — it landed with
   `appPageID: null`, which the page-specific resolver then filters out. Reproduced live: a widget
   appeared correctly in the Section Details widget list (proving the placement was created) but
   never rendered on canvas, even after a full hard reload. Fixed in `CanvasPanel.tsx`'s drop
   handler by stamping `activePageID` whenever the target section is the primary one.

**Separately**: `RouteResolverService.resolve({})` (called with no widgetID/formID/pageID — exactly
what a bare app-root Preview URL like `/{appID}?tenantID=...` with no `/page/{slug}` segment
produces) used to fall through to picking `paneWidgets[0]` — an ARBITRARY widget by raw
`displayOrder` across every page — with no concept of "this app's own home page," instead of
consulting `AppPage.isDefault`. If that first-by-displayOrder widget happened to be an orphaned
`appPageID: null` leftover (e.g. from an earlier abandoned drag-drop test), the app's real home page
content was invisible on its own bare Preview link while working correctly when a specific
`/page/home` URL was used — a genuinely confusing split-behavior bug reproduced live on two separate
apps. Fixed by preferring `AppPage.isDefault` over the arbitrary-widget fallback when no anchor is
given at all.

**Lesson: "no page/widget specified in the URL" and "correct page targeted" are NOT the same
resolution path in this engine, and the gap between them silently defaults to something arbitrary
rather than the app's own configured home page.** Always test an app via its bare root Preview URL,
not just via explicit `/page/{slug}` deep links — they can (and did) produce different, silently
wrong results from the identical underlying data.

## 2026-09-10/11 — Native HTML5 drag-and-drop in the App Studio Designer cannot be driven by browser automation, including synthetic DragEvent dispatch

Multiple approaches tried and confirmed non-functional for placing a widget via canvas drag: (1) a
plain simulated mouse drag (mousedown/move/up sequence), tried twice with different coordinates —
no widget added, no console activity at all; (2) manually dispatching a full synthetic `DragEvent`
sequence (`dragstart`/`drag`/`dragenter`/`dragover`/`drop`/`dragend`, sharing one `DataTransfer`
instance, fired via `document.elementFromPoint` + `dispatchEvent` in page-context JS) — also
produced zero effect and zero console signal that the app even attempted to handle it. A REAL human
mouse drag in the same browser, same session, DID work correctly (confirmed by the app author doing
it live). This strongly suggests a pointer-based (not native-HTML5) drag library, or one gated on
trusted-event checks, rather than a bug in the Designer itself — but it means **automated testing of
the Designer's canvas drag-to-add flow is not currently possible**; any test plan for this surface
needs either a real human driving it, or should route through the click-based alternative below.

**Workaround that IS automatable and equally real**: `AddWidgetModal` (the "+ Add Widget" button)
already accepts and correctly threads through `appPageID` end-to-end — but the button that opens it
(`ThisSectionAddWidgetButton` in `SectionDetailsContent.tsx`) used to be hidden entirely whenever
the section was the page-controlled primary one (`{!readOnlyWidgets && <ThisSectionAddWidgetButton/>}`),
leaving NO reachable way to add a widget to real page content without a working drag gesture. Fixed
by always rendering the button and passing `activePageID` through when the section is primary — this
is now a fully click-based, automatable, and (per the earlier finding above) it's also just a better
UX than requiring drag for the single most common authoring action. The same read-only section also
had no way to REMOVE a widget from it (`WidgetChip.tsx`'s `readOnly` branch renders a plain
button-less `<div>`), fixed by adding a "Remove from section" action to the Widget Details panel
(the one place every widget — read-only chip or not — is reachable from), calling the store's
existing `removeAppWidget` (a true disassociate: soft-deletes the placement, leaves the shared
Widget definition intact for reuse elsewhere).

## 2026-08-30 — Missing-controller bug that looked like a routing/middleware mystery was actually a project-convention miss, confirmed by a live route-table dump

New anonymous public-asset GET endpoints returned bare, empty 404s despite correct DB state and a
successful full rebuild — repeatedly. The natural hypothesis ("SPA-fallback/static-file middleware
intercepting anonymous GETs before MVC routing reaches them, since every existing anonymous endpoint
in this codebase happens to be POST") was reasonable and worth checking first, but wrong — confirmed
via a temporary `IActionDescriptorCollectionProvider` route-table dump that the controller's routes
were **completely absent from the registered route table**, and via checking `Program.cs` directly
that no `UseStaticFiles`/`MapFallback`/`UseSpa` middleware exists in this pipeline at all.

**Real cause**: this codebase has an established convention — never `ProjectReference` a domain's
`.Api` project into the platform WebApi; instead **copy the controller source file** into
`BizFirst.Ai.Platform.Web.Server.Core\Controllers\...` (already true of `DocumentController.cs`/
`PublicShareController.cs`). The new controller had been created only in the `.Api` project and never
copied — invisible to the running app regardless of how many times it was rebuilt. This also
explained why the *other* new endpoints (`publish-public`/`unpublish-public`) worked the whole time —
those are actions on the already-copied `DocumentController`, not a new file.

**Lesson: when a brand-new endpoint 404s cleanly (not a 500, not a timeout) despite a clean rebuild,
check whether the controller file physically exists in every project this codebase's own convention
says it must be copied into — before chasing middleware order.** A live route-table dump settles this
definitively in one step; don't guess from symptoms when the actual registered-routes list is one
temporary DI call away.

## 2026-08-30 — Memory-constrained dev box (~7.7GB RAM): two independent, additive causes of the same "hung fetch" symptom

Repeated, real symptom: the Designer's canvas got permanently stuck on a loading state, even after
every underlying API call had actually resolved 200. Two genuinely separate root causes were found
and fixed, both worth knowing for any future work on this box:

1. **`fetchJson()` (`app-studio-api-client-js/src/http.client.ts`), used by every app-studio API
   call, had zero client-side timeout.** When the backend genuinely stalls under memory pressure (not
   crashes — stalls), the underlying `fetch()` promise just hangs forever; nothing ever resolves or
   rejects, so a loading spinner with no error is exactly what a user sees, indistinguishable from a
   real infinite hang without checking network state directly. Fixed with
   `AbortSignal.timeout(30_000)`, matching the existing `HttpClient` convention already used
   elsewhere in this codebase (a fix that generalizes — every call gets it, not a one-off patch).
2. **Separately, 11 leftover MSBuild build-server nodes (`dotnet.exe ... MSBuild.dll ...
   /nodemode:1 /nodeReuse:true`) accumulated from repeated `dotnet build`/rebuild cycles**, each
   holding ~25-33MB, ~350MB total — silently never released between builds. This machine's own
   documented constraint already says every .NET build here needs `-nodeReuse:false` for exactly this
   reason; a build invoked without it (the default) leaves these idle nodes behind. **They are safe to
   kill at any time** (`Stop-Process -Force` on any `dotnet.exe` whose command line contains
   `MSBuild.dll...nodemode` — never touch the one real running WebApi process, identifiable by its
   command line being the actual `.dll` entry point, not `MSBuild.dll`) — freed ~350MB and measurably
   stabilized the WebApi's ability to respond at all.

**Lesson: on this box, before diagnosing a "hung"/slow backend as a code bug, check (a) whether the
specific client involved has a timeout at all, and (b) `Get-Process dotnet` for accumulated idle
MSBuild nodes** — both are real, independent, additive contributors to the same visible symptom, and
neither is a code defect in the feature you're actually testing.

## 2026-08-30 — Schema drift between the live dev DB and the declarative SSDT source happened THREE separate times this session, in both directions

Not one incident — a recurring pattern worth internalizing as a standing check, not a one-off fix:

1. A `PUBLISHED_VIDEO`-related set of columns (`PublishStatus`/`YouTubeVideoID`/`DurationSeconds`)
   was declared in `dbo\Tables\Doc_Documents.sql` (the SSDT source of truth) but missing from the
   live DB — a completely unrelated pre-existing feature, discovered only because a generic
   `Doc_Documents` query selects those columns unconditionally regardless of which document type is
   being queried, so ANY document query broke, not just that feature's own.
2. Four whole tables (`Doc_DocumentContents`, `Doc_DocumentShares`, `Doc_DocumentShareAccessLogs`,
   `Doc_DocumentComments`) were declared in SSDT but didn't exist live at all — found via real
   `Invalid object name` errors while testing an unrelated app that happened to touch
   `document-manager`'s sharing/comments features.
3. **The reverse direction**: new columns this session's own work added (`Doc_Documents`'
   `IsPublicAsset`/`PublicAssetKey`/`PublicAssetPublishedOn`) were applied to the live DB via a
   migration script but the SSDT declarative source (`Tables\Doc_Documents.sql`) was never updated to
   match — caught by an independent code review, not caught live (a fresh environment built from the
   SSDT project alone would have silently been missing these columns).
4. A migration for an entirely different, unrelated feature (`AIExt_AppPages.SEOConfiguration`, from
   this same session's own earlier work) sat written-but-unapplied for a long stretch and silently
   broke a real page-list feature the moment a real query finally exercised that column, well after
   the code that needed it had already shipped and been reported "done."

**Lesson: schema drift is bidirectional and doesn't announce itself — it waits for the first real
query that happens to touch the missing/undeclared piece, which can be minutes or hours after the
actual code change.** After writing ANY new migration: (a) apply it to the live dev DB immediately,
don't leave it "written for review" indefinitely if the dependent feature is already live, and (b)
separately verify the SSDT declarative `Tables\*.sql` source was updated to match, so a *fresh*
environment wouldn't silently diverge from this one. Do both checks every time, not just one.

## 2026-08-30 — Real Critical bug: an immer-patch undo stack coexisting with untracked mutations, with no invalidation between them, can resurrect deleted data

Confirmed by an independent adversarial code review, then fixed. The undo/redo feature's own design
doc had explicitly scoped only two mutation types as "safe" (pure client-side layout edits, zero
network calls) and left five others ("Category B" — delete/reorder/rename/widget-config-save, all of
which fire real API calls) deliberately untracked. That scoping was correct on its own — but nothing
stopped an already-queued Category-A undo patch (captured against an OLDER shape of the shared
`layout` state) from being replayed AFTER an intervening, untracked Category-B mutation had already
changed that same state. Concrete repro: move a section (tracked) → undo it → delete a DIFFERENT
section (untracked, cascades real backend widget deletes) → redo the original move — the stale
patch, still containing the now-deleted section, silently resurrects it client-side with widgets
already gone server-side; saving afterward persists a dangling, orphaned section with no recovery
path.

**Fix**: every untracked (Category B) mutating action now also clears the undo/redo stacks in its own
state-update call — an out-of-band edit invalidates any pending undo/redo history rather than being
allowed to interact with it. Deliberately coarse (loses undo history on any untracked edit) rather
than attempting to rebase/patch-merge across the untracked edit, which would be far more complex and
itself risk a subtler version of the same corruption.

**Lesson: any undo/redo (or similarly stateful, patch-based) system that only tracks a SUBSET of
mutations must explicitly invalidate its history on every mutation OUTSIDE that subset, not just
scope the subset correctly and assume the untracked ones are "safe by omission."** Untracked doesn't
mean harmless to the tracked stack's own assumptions.

## 2026-08-30 — Real, hard security correction: public asset URLs must never be keyed by an internal sequential ID

Applied mid-build, before any code shipped against the wrong model. The original working assumption
for a new "digital assets library" feature (public/private file hosting) was headed toward serving
public files by `DocumentID` — an internal, sequential, auto-increment primary key. Two independent
problems with that: (1) it's enumerable — anyone can walk `?id=1,2,3...` and discover every public
(and, if the endpoint doesn't gate correctly, private) asset in the system; (2) it requires a live
database lookup on every single page view of every embedded image, which doesn't scale for a
public-facing, high-traffic site (1M page views × 10 images/page = 10M DB-backed lookups).

**Fix, matching how every real media-hosting platform does this** (Wix, Cloudinary, imgix, S3-backed
CDNs generally): mint a genuinely opaque, non-sequential storage key (GUID-shaped, matching this
codebase's own existing key-generation convention elsewhere) at publish time — the database is
touched exactly once, to mint the URL and record metadata, never again per view. The internal
sequential ID stays purely internal (admin/management APIs only) and must never appear in a
public-facing URL.

**Lesson: for ANY future "make X publicly servable" feature in this codebase, default to an opaque
key minted at publish time, never the record's own primary key, and design the DB-touch to happen
once at publish time, not per view** — this is now the established, correct pattern, not a one-off
design choice for this feature.

## 2026-08-30 — Cross-agent file collisions are real during parallel background-agent work, not just a theoretical risk

Multiple background agents building adjacent features concurrently DID collide on shared files
(e.g. two different agents both landing edits in `DesignerToolbar.tsx`/`CanvasPanel.tsx` around the
same time) — resolved correctly by the agents themselves (verified their own changes survived the
other's concurrent edit), but this was closer to a real problem than planned, not just a theoretical
risk that never materialized. Separately: two agents' live-browser-verification steps interfered with
each other by sharing one Chrome tab — one agent's UI state (a panel it had open) showed up in
another agent's screenshot, producing a false-negative "verification inconclusive" result that had
nothing to do with either agent's actual code.

**Lesson: (1) sequence parallel work by ACTUALLY TRACED file overlap between each task's own design
doc, not by "run everything in parallel and hope" — a coordinator reading each design doc's own
"files touched" section before dispatching catches most of this. (2) Every agent doing live browser
verification while other agents may also be testing concurrently needs its own dedicated browser
tab, created fresh, never a shared/reused one — this is cheap to require and expensive to debug
around after the fact.**

## 2026-08-30 — `@passport/fake-auth`'s real limits, and the standing rule around real credentials

The sanctioned dev-only fake-auth localStorage-seeding bypass (see `architecture.md` §5 for the exact
recipe) is genuinely useful for isolated frontend-only testing, but seeding it on the wrong origin
(or leaving it seeded when a real backend round-trip was actually needed) produced a confusing false
symptom — an app showing an empty "no apps yet" state that looked like a data bug but was actually
"the authenticated identity is fake and has no real data." **It is never a substitute for a real
login when the test actually needs real backend authorization to succeed** (a fake token always
401s on a real authenticated call) — and separately, real user sessions kept dropping under this
session's backend memory pressure because `verify-token` round-trips timed out before completing,
which the frontend correctly (if confusingly) treated as an invalid session and bounced to the real
login page — this looked like a login/auth bug on first glance but was actually a backend-latency
symptom, confirmed by checking that the WebApi process itself was still alive and listening, just
slow.

**Standing rule, not App-Studio-specific**: never enter a real password into any login field, under
any circumstance, even with explicit user permission to do so. When a real session is needed and
none exists, ask the human to log in — don't work around it, don't guess whether "just this once" is
fine.
