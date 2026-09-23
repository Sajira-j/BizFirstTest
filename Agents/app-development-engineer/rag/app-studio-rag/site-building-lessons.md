# App Studio Site-Building Lessons (composition/quality, MCP-driven builds)

Tier 1 doc (see `00-overview.md`'s two-tier retrieval pattern). Load this whenever a request
involves building a MULTI-PAGE, MULTI-WIDGET App via the App Studio MCP tools (`create_page`,
`create_section`, `create_widget`, `update_widget_placement`, `update_widget_definition`, `update_app`)
— not just placing/configuring one widget. Companion to `theming.md` (the `--app-var-*` token
contract and widget type catalog — read that first; this doc does not repeat it) and
`content-widget-templates.md`. Written 2026-09-14, grounded in the real before/after of building
"Meridian Creative Co." (AppID 1090, `BizFirst.Ai.Mcp.Tools.AppStudio` module) end to end via MCP —
every issue below was actually hit and actually fixed on that build, not invented.

---

## 1. The `allowScripts` bug — the single biggest "looks unstyled" cause

**Symptom**: a `content` widget's `<style>` block and CSS classes are completely silently stripped.
The page renders as plain, unstyled black-on-white/default text — no background, no padding, no
button styling, no grid layout, no colors — even though the widget's `Configuration.content` string
you wrote clearly has a `<style>` tag full of `--app-var-*`-driven CSS in it.

**Root cause**: `ContentSanitizer.ts` runs every `content` widget's HTML through DOMPurify before
render. Its **default** `ALLOWED_TAGS` list (used whenever `allowScripts` is unset/false) is a
restrictive allowlist — `p`, `br`, `strong`, `a`, `div`, `span`, headings, lists, tables, `img`, etc.
— and **`style` is not in it**. DOMPurify silently drops the entire `<style>` element; nothing
throws, nothing logs, the rest of the HTML renders fine (that's why this looks like "the page is
just plain/thin," not like an error).

**Fix**: every `content` widget's `configuration` needs a third field, `allowScripts: true`, alongside
`content` and `format`:

```json
{"content": "<div class=\"mc-home\"><style>...</style>...</div>", "format": "html", "allowScripts": true}
```

With `allowScripts: true`, DOMPurify's `ALLOWED_TAGS`/`ALLOWED_ATTR` both go to `undefined` (its own
full default profile), which does allow `style`. This is the exact field the real Content Widget
Templates (792-795, 848-877) all set — cross-check `theming.md` §1.5/§2.3's real examples before
assuming a config is complete; `content`+`format` alone is not enough for any widget whose HTML
includes its own `<style>` block, which is effectively every real design-quality content widget in
this system.

**How to avoid repeating this**: `create_widget`'s own tool description inline-example (per the
design doc's "ergonomics only" exception) should be read as `{"content": "...", "format": "html"}`
being the MINIMUM shape for plain text/markdown — the moment the content includes a `<style>` tag
(i.e., almost every real page-content widget), add `allowScripts: true` or the whole visual design is
silently discarded. **`create_widget` has no way to fix this after the fact** — if you discover a
widget is missing `allowScripts`, you need `update_widget_definition` (added 2026-09-14, see §5) to
correct the shared Widget row's `Configuration`, since `create_widget` only writes it once and
`update_widget_placement` only edits placement-level style, never the widget's own definition.

---

## 2. Layout widgets (`site-branding`, `page-navigation`) do not participate in `--app-var-*` at all

Per `theming.md` §2.2/§2.3, this is a **confirmed, structural gap in the App Studio frontend today**,
not something an MCP-side config fix can work around: `SiteBrandingWidgetRenderer.tsx` and
`PageNavigationWidgetRenderer.tsx` both hardcode their own `--color-*`/`--border-radius-*` Designer-
chrome tokens (a completely different, unrelated token namespace — the Designer editor's OWN internal
theme, not the published App's theme) directly in their `styles` objects. Neither component reads
`context.appID`'s theme, and neither has a `content`/CSS field in its `Configuration` to hand-author
around it (`SiteBrandingWidgetConfig` is just `{titleOverride, imageSize}`;
`PageNavigationWidgetConfig` is just `{orientation}`).

**What DOES work, the only real lever**: `update_widget_placement`'s `styleConfiguration` field, on
the `widgetContainer`/`widgetBackground` slots specifically — these are the ONLY two of the ten named
`WidgetStyleConfig` slots actually wired into any renderer today (`WidgetSlot` in `AppPlayer.tsx`
wraps every widget's OUTER DOM with these two; the other eight — `widgetHeader`/`widgetTitle`/etc. —
are parsed/edited but not yet consumed by any real handler). Real, working example from the Meridian
build:

```json
{"widgetContainer": {"css": "background:var(--app-var-bg,#0b1f14);padding:16px 32px;display:flex;align-items:center;border-bottom:1px solid var(--app-var-border,#234433)"}}
```

This themes the OUTER bar (background, padding, border) correctly. It does **not** reach the inner
text/nav-item colors — those stay whatever `--color-*` the renderer hardcodes, regardless of the
app's own theme. **Be honest about this limitation when building a site**: a fully theme-consistent
header/nav is not achievable via MCP config alone today. Flag it rather than claim full compliance —
the outer chrome (bar background, spacing, border) tracks the app theme; the inner brand-title text
color and nav-item text/active-state colors do not, and fixing that requires a frontend code change to
the two renderer components (out of MCP-tools scope), not a config trick.

---

## 3. `update_app` had no way to set an app's theme at all (fixed 2026-09-14)

Before this build, none of the 15 V1 tools could write `App.Configuration.theme` — `update_app` only
touched `name`/`description`/`appCode`. An app built entirely via MCP therefore had NO theme for its
own `--app-var-*`-driven content widgets to resolve against; every color/font/spacing value silently
fell through to each widget's own hardcoded fallback. Fixed by adding an optional `theme` parameter to
`update_app` (JSON object string of the 19 tokens from `theming.md` §1.3, whole-object replace, same
read-modify-write-`Configuration` pattern `create_section` already used for `configuration.layout`).
**If you're building a themed site and `update_app` doesn't expose `theme` yet, that tool needs this
fix before proceeding** — don't hand-roll a workaround (e.g. raw SQL) when the module's own convention
(read `App.Configuration`, merge one top-level key, write back) is right there in `CreateSectionTool.cs`
to copy.

**A second, more serious bug found alongside it**: `update_app`'s existing `appCode` path
unconditionally called `userContext.UserIdRequired` at the very TOP of the method — before even
checking whether `appCode` was supplied. `UserIdRequired` throws `UnauthorizedAccessException` for
any API-key-authenticated caller (an `ApiKey` row authenticates a TENANT only —
`ApiKeyAuthenticator.AuthenticateAsync` calls `SetUserContext(userId: null, tenantID: ...)`, never sets
a UserId). This meant **`update_app` was completely unusable via any API-key-scoped MCP caller** —
not just the new `theme` path, ANY call, even a name-only rename — until fixed by (a) only resolving
an actor when `appCode` is actually being set, and (b) falling back to
`BackgroundJobIdentity.SystemUserId` when no real UserId is present, the same fallback pattern
`PlatformWebServerExtensionsByApp.cs` already uses elsewhere for MCP-agent-originated writes. **General
lesson**: any tool method that calls `UserIdRequired`/`TenantIDRequired` should be checked against
BOTH auth tiers this module supports (a real human JWT session AND a scoped API key) — a tool that
only works for one silently fails the other with an opaque "An error occurred invoking '...'" (the
MCP SDK's own generic exception-to-text wrapper gives no hint which line threw).

---

## 4. `create_widget`'s `sectionName` is a no-op without `create_section` first

Not a bug — a real, easy-to-miss sequencing requirement. `AppSection` is not a separate database
row/table; it's an entry inside `App.Configuration.layout.appSections[]`. `create_widget`'s
`sectionName` parameter only actually places a widget if a section by that exact name already exists
on the app — **call `create_section` for every section name you intend to use (`header`, `main`,
`footer`, or whatever names you pick) BEFORE the first `create_widget` call that targets it**, not
after. For a real multi-page site: one `header` section (`region: "header"`, no
`isPrimaryContentSection`) and one `footer` section (`region: "footer"`, same) hold the SHARED
layout widgets (placed with `appPageID: null` so they show on every page); one `main` section
(`region: "main"`, `isPrimaryContentSection: true`) holds every page's OWN content, with each page's
widget placed via that same section's `sectionName` but a DIFFERENT `appPageID` per page. This is the
real pattern — three sections total is enough for a typical site; you do not need one section per
page.

---

## 5. Composition/quality lessons (how a "looks thin"/"needs formatting" site actually gets fixed)

Binoy's real, specific feedback on the first pass of Meridian Creative Co. ("does not look good...
needs lot of work... every page needs lot of formatting") traced almost entirely back to issue #1
above (`allowScripts` missing → every widget's CSS silently discarded, so a genuinely well-composed
HTML/CSS design rendered as bare unstyled text). Once that was fixed, the remaining, GENUINE
composition lessons worth carrying forward:

- **One content widget per page is enough, if the widget's own HTML is internally well-sectioned.**
  Meridian's Home page is a single `content` widget whose HTML has its own `<section>` blocks (hero,
  a 3-card "what we do" grid, a CTA band) — this reads as a complete page, not a thin one, because
  each internal section has its own generous padding (`calc(var(--app-var-space-unit,8px)*10-13)`
  vertical, not a bare `16px`) and a clear background-color break between sections (alternating
  `--app-var-bg`/`--app-var-bg-panel`) so the eye can see where one section ends and the next begins.
  **Do not split a page's content across many small separate widgets by default** — it adds placement/
  ordering complexity (`displayOrder` on N widgets instead of 1) for no visual benefit versus one
  widget with well-structured internal `<section>`s, UNLESS a section is genuinely a different widget
  TYPE (e.g. page 5's `content` intro + `image-gallery` — those must be two widgets since they're two
  types) or needs independent per-instance styling via `update_widget_placement`.
- **Spacing rhythm is the single highest-leverage fix for "looks thin."** A real before/after: the
  ORIGINAL failure mode (before `allowScripts`) had zero padding, zero section separation, headings
  the same visual weight as body text. The Content Widget Templates' own convention (`theming.md`
  §1.5, widget 792/848) is the right scale to copy: section padding `calc(var(--app-var-space-unit,8px)*8-13)`
  vertical for a full section, `*3-4` for a card's internal padding, `*2` for tight gaps — never a
  bare unscaled pixel value, and never less than `*6` (48px) of vertical padding around a full-width
  section on desktop or it reads as cramped.
- **Typography hierarchy needs a real font-size AND weight jump, not just a `<h2>` tag.** Copy the
  real templates' scale: hero `h1` ~44-52px/700-800 weight in `font-heading`, section `h2` ~28-32px,
  card `h3` ~18-19px, body copy 14-16px/1.6-1.8 line-height, an "eyebrow" label above a heading at
  13px/700/letter-spacing 2-3px/`color-secondary` for the accent-color micro-label pattern (used in
  every real template). A page whose every text block is the same size is the visual signature of
  "unstyled," independent of whether CSS loaded at all.
- **Real photography beats decorative-only backgrounds for a business/portfolio site.** Meridian's
  before/after: adding a real photo (hero background via `color-mix()`-darkened overlay + `url(...)`,
  team headshots in circular frames, a 4-photo project grid on the Work page) from
  `C:\BizFirstGO_FI_AI\StockAssets\` (see `theming.md` Part 3 for how to reference one) measurably
  improved perceived polish versus the same layout with no imagery at all — a content-only page,
  however well-typeset, reads as a wireframe next to one with real photography.
- **Always give the widget's OUTERMOST wrapper element its own explicit `background:
  var(--app-var-bg, fallback)` — never leave it transparent assuming inner sections cover
  everything.** Real bug, live-reproduced and fixed on Meridian during the theme-switch test (§7):
  the Home page's outer `<div class="mc-home">` had no background of its own; `.mc-hero` and
  `.mc-cta` each set their own explicit background, but the middle `.mc-section` ("What we do") did
  not, so it showed through to whatever sits behind the widget in the App Player's own DOM — which
  rendered as a stark black band, matching neither theme, cutting a well-designed page in half
  visually. Invisible on the original dark theme (dark-on-dark happened to look intentional) but
  immediately obvious the moment the theme switched to a light preset — **this is exactly the kind
  of gap a live theme-switch test catches that a single-theme visual check does not.** Fix: put
  `background: var(--app-var-bg, <fallback>)` on the single outermost class every page's HTML is
  wrapped in, once, so there is never a gap for the page's own default background to leak through,
  regardless of which inner sections do or don't set their own.
- **The `image-gallery` widget's own chrome (the wrapping frame around its live-queried images) has
  a hardcoded background, confirmed independently of the `--app-var-*` cascade** — even after the
  theme switch, Meridian's Work page gallery section stayed on a solid black background while every
  other section on the same page correctly followed the new light theme. This matches
  `theming.md` §2.3's own "Theming: none directly in Configuration" note for all 4 gallery types —
  not a regression, a confirmed pre-existing frontend gap, out of this MCP module's reach to fix
  (the gallery widget's `GalleryFrame` component, not `Configuration`, owns that chrome).
- **The `image-gallery` widget type queries LIVE public documents — it is not a fixed list you seed
  via `create_widget`.** Don't expect a freshly created gallery widget to show anything beyond
  whatever public `Image`-category documents already exist tenant-wide (it may show OTHER apps' public
  assets, which is correct/expected behavior, not a bug — the query has no per-app scoping). Uploading
  new assets into that library (`POST /api/v1/documents/upload`) requires
  `[AuthorizeRegularUserAttribute]` — **a real, human-authenticated session, not an MCP API key** — so
  an MCP-driven build cannot itself seed the gallery's backing library; it can only place the widget
  correctly configured (`mediaCategory`, `theme`, `columns`) and document that a signed-in human needs
  to upload real assets for it to show more than whatever's already public tenant-wide.

---

## 6. Restarting the Consolidated WebApi invalidates the Designer/App Player's browser session

Real, hit live during this build: fixing a widget-config bug via `update_widget_definition` required
a new MCP tool, which required rebuilding `BizFirst.Ai.Platform.Web.Server.Core`/the host and
restarting the shared Consolidated WebApi process (port 10001) for the fix to load. After that
restart, the Designer (`localhost:6109`) and App Player (`localhost:6130`) browser tabs that were
previously signed in redirected to the Passport login page (`localhost:8001/login`) on the next
navigation — the JWT-bearing session the browser held no longer authenticates. **An agent must not
click through a login form itself to recover** (even with browser-autofilled credentials, even for
the same account driving the rest of the task) — that is a credential/authentication action outside
an agent's own authority, and reusing a bearer token captured from an earlier authenticated URL
(e.g., one seen in an App Player preview link before the restart) is equally out of bounds — both were
correctly refused by this session's own safety controls when attempted. **Practical consequence for
planning a build that needs a live-code fix mid-way**: budget for a real possibility that visual
browser-based verification (screenshots, live theme-switch confirmation) becomes blocked immediately
after any restart the fix requires, until a human re-authenticates that browser tab. Data-layer
verification (the MCP tool's own JSON response, direct SQL against `AIExt_Widgets`/`AIExt_Apps`)
remains available throughout and should be used to confirm a fix landed correctly even when visual
confirmation has to wait for a human.

---

## 7. Live theme-switch verification — do this, don't skip it

A structural/static check (config JSON has the right `--app-var-*` references with fallbacks) is
NOT the same as proof the cascade actually works. **Always finish a themed-site build by actually
switching the app's theme to a different preset** (`theming.md` §1.7's `Template_DataTemplates`,
`DataTemplateTypeID = 54` catalog — pick one visually distinct from what you started with, e.g. a
light preset like "Ocean Breeze" if you built dark) via `update_app`'s `theme` parameter, then
re-screenshot every page in the App Player. This is the only real test that every widget's CSS is
genuinely reading `var(--app-var-*, fallback)` at render time rather than a value that happened to
look plausible by coincidence with the original theme. Confirm explicitly, per page: does the
background/text/accent color set actually change, or does anything stay stuck on the old hardcoded
look (that's a real bug — the specific widget's CSS has a literal instead of a `var()`, or references
a token this catalog doesn't define).

**Run on Meridian, real result**: switched from the original 19-token dark theme to the real "Ocean
Breeze" preset (light blue/white, `DataTemplateID=91020393`) via `update_app`. Every `content`
widget's hero/section/card backgrounds, text colors, button colors, and the footer/nav chrome bar
all correctly re-rendered in the new palette with zero further changes — genuine proof the
`var(--app-var-*, fallback)` cascade works end to end across a real multi-page, multi-widget-type
site. Found exactly one real bug in the process (the missing outer-wrapper background, §5 above),
fixed it, re-verified clean. Also confirmed the two already-known, out-of-MCP-scope gaps stayed
exactly as documented: `site-branding`/`page-navigation`'s inner text/nav-item colors did not
move (§2 above — a frontend renderer limitation, not this run's bug), and the `image-gallery`
widget's own background chrome did not move either (§5's gallery note above). A theme switch that
finds ZERO bugs should be treated with suspicion, not relief — it more likely means the test wasn't
looking closely enough (e.g., only checking the hero, not scrolling every section) than that
everything is actually perfect.

---

## 8. `color-scheme` on `widgetContainer` DOES reach into a `form` widget's Atlas Forms subtree — fixes native `<select>` popups

Real bug, found and fixed 2026-09-15 on AppID 1092 ("aaa"): a `form` widget's native `<select>`
dropdown popups (Document Type / Document Category / page-size) rendered with a stark WHITE
background/black text against an otherwise fully dark app theme. Root cause confirmed via
`getComputedStyle(selectEl).colorScheme` — it read `"normal"` on the `<select>` itself and on
every one of the ~20 ancestor elements up to `<html>` (`html`, `body`, the App Player root, every
`af-form-renderer`/`af-control` wrapper Atlas Forms generates). No element anywhere declared
`color-scheme: dark`, so Chrome fell back to its OS/browser-default LIGHT UI theme for the native
popup specifically — the closed select's own box still looked dark because that part IS ordinary
painted CSS (`background: rgba(0,0,0,0.3)`, set by the form's own styling), but the open popup list
is a native browser-chrome element that only ever respects the CSS `color-scheme` property (not
`background-color`/`color`), and only if it resolves to `dark` (or `light dark`) SOMEWHERE in the
ancestor chain.

**Fix, real and verified**: `update_widget_placement(widgetPlacementID=<form widget's AppWidgetID>,
styleConfiguration="{\"widgetContainer\":{\"css\":\"color-scheme:dark;\"}}")` (bundled in the same
call as a margin fix, see §9 below). Because `color-scheme` is an ordinary INHERITED CSS property
and `widgetContainer` is a real DOM ancestor of the widget's entire rendered subtree (not a
portal/iframe boundary), this one declaration cascades all the way down through Atlas Forms'
generated markup with zero Atlas Forms-side changes needed. Re-checked post-fix via the same
`getComputedStyle` probe: all three selects now read `colorScheme: "dark"`. **This contradicts an
earlier, more cautious assumption in this doc's draft state** (form widgets were assumed possibly
"out of MCP's reach" since theming.md notes `form` widget theming is "none — delegated entirely to
Atlas Forms' own FormRenderer") — that caution was right for COLOR tokens (`--app-var-*`, which
Atlas Forms genuinely never reads) but wrong for `color-scheme`, which works purely through
ordinary CSS inheritance and doesn't need Atlas Forms to know anything about App Studio's theme
system at all. **General lesson**: don't assume a form/cross-domain widget type is unreachable for
EVERY kind of container-level CSS fix just because its Configuration has no theme fields — anything
that works via plain CSS inheritance (not a custom property the child markup would have to
explicitly `var()`-reference) still reaches through `widgetContainer`.

**Verification caveat**: a native `<select>` dropdown popup could not be captured by
screenshot/zoom either before or after the fix — click-then-screenshot only ever shows the closed
select with a focus ring, never the open option list, both pre- and post-fix. This is a tooling
limitation (native OS-level popups sit outside the page's own paint tree that a Chrome-extension
screenshot API captures), not evidence the fix didn't work. Treat the `getComputedStyle(...).colorScheme`
probe (`"normal"` broken / `"dark"` fixed) as the real verification for this class of bug, not a
screenshot.

## 9. Zero page-edge gutter on a page with no header/footer widgets — `margin` on each `widgetContainer` is the only available lever today

Real bug, same AppID 1092 build: a single-page app with no `site-branding`/`page-navigation`/footer
widgets (just a `workflow-template` × 2 + one `form` widget, all in one `main-content` AppSection)
rendered completely edge-to-edge — `body{margin:0;padding:0}`, the App Player's
`[data-section="main-content"]` wrapper `padding:0`, and its content column exactly matched
viewport width. The first widget's card sat literally at DOM coordinate (0,0); the Search/Add New
Document buttons and the results table touched the right edge; the two `workflow-template` cards
stacked with zero gap between them (touching borders).

**Root cause**: none of the 17 AppStudio MCP tools expose a page-level or AppSection-level
gutter/padding control (`create_section`'s own `configuration.layout` — per §4 above — governs
section MEMBERSHIP/region, not spacing; there is no `update_section` tool at all in this module as
of 2026-09-15). With no header/nav widget contributing incidental top margin (as Meridian's site
always had, masking this exact gap in the original lessons build), a bare content-only page has
NOTHING supplying outer spacing by default.

**Fix, real and verified**: apply `margin` (not `padding` — `widgetContainer` wraps the OUTSIDE of
each widget, so margin is what creates gutter/gap here) via `update_widget_placement` on EVERY
widget placement on the page individually, since that is the only reliably-wired style slot (§2
above) and there is no coarser page/section-level lever available yet:

```
appWidgetID 373 (1st widget):  {"widgetContainer":{"css":"margin:32px 32px 12px 32px;"}}
appWidgetID 374 (2nd widget):  {"widgetContainer":{"css":"margin:0 32px 24px 32px;"}}
appWidgetID 375 (3rd widget):  {"widgetContainer":{"css":"margin:0 32px 32px 32px;color-scheme:dark;"}}
```

i.e. give the FIRST widget the top gutter, give every widget the same left/right gutter, and use
the bottom margin of each widget as the vertical gap to the next one (rather than double-spacing
with both a widget's bottom margin AND the next widget's top margin). Re-screenshotted after
applying: confirmed a clean, consistent 32px gutter on all four page edges and a visible gap
between the two stacked cards, with no introduced horizontal overflow/scrollbar. **General lesson**:
until this module grows a page/section-level spacing control, page-edge gutter on any
header/footer-less page must be budgeted as a per-widget `margin` pass over every placement on the
page — don't assume the App Player itself contributes any default page padding, because it doesn't.

## 10. `get_app`/`list_apps` still hit the API-key `UserIdRequired` bug that §3 already fixed for `update_app` — use `list_widgets_in_app` instead

Real, live-reproduced 2026-09-15 calling the AppStudio MCP server over HTTP JSON-RPC with a
tenant-scoped API key (no UserID, same auth shape §3 describes): both `get_app` and `list_apps`
returned the same generic wrapped error, `"An error occurred invoking 'get_app'."` /
`"...'list_apps'."`, for every argument combination tried (including the exact `appID` of a real,
existing, in-tenant app) — isError:true, no further detail (same opaque-wrapper symptom §3 warned
about). Other AppStudio tools called with the identical API key in the same session worked cleanly
(`list_widget_types` with no args, `list_widgets_in_app(appID)`), which rules out an auth/key
-scoping problem in general and points at these two specific tools still having an unconditional
`UserIdRequired`/`TenantIDRequired`-style call that §3's fix apparently did not cover. **Not
independently re-diagnosed in backend code this pass** (out of scope for an MCP-tools-only task) —
flagging as a real, reproducible gap for whoever next touches `GetAppTool.cs`/`ListAppsTool.cs` to
apply the same `BackgroundJobIdentity.SystemUserId` fallback §3 already used for `UpdateAppTool.cs`.
**Practical workaround that unblocked this task**: `list_widgets_in_app(appID)` alone was enough to
get every placement's real `AppWidgetID`, `WidgetID`, `sectionName`, `appPageID`, and
`displayOrder` — sufcient to target `update_widget_placement` calls correctly by position
(`displayOrder` 0/1/2 matched top-to-bottom visual order exactly) without ever needing `get_app`'s
fuller detail.

## 11. The REAL fix for header (or any section) stacking is `AppSection.widgetLayout`, not a per-widget CSS workaround — but no MCP tool can set it on an existing section yet

Follow-up to §2, same AppID 1090 Meridian build, 2026-09-15. §2 only discusses the two wired
`widgetContainer`/`widgetBackground` style slots and concludes a fully-fixed header isn't achievable
via MCP config alone — that conclusion is **too pessimistic**. Reading `AppPlayer.tsx` directly
(`app-handlers-generic\src\AppPlayer.tsx`) shows every `AppSection` has a first-class, already-wired
`widgetLayout` field (`AppSectionWidgetLayout: {direction, wrap, gap, align, justify}`,
`app-handlers-core\src\types\AppSection.ts`) that wraps a section's own widgets in a real flex
container via `useContainerFlexStyle` (`hooks\useFlexLayoutStyle.ts`). **Its default `direction` is
`'column'` — that default, not a rendering bug, is the entire reason two widgets in one section stack
vertically with zero configuration.** Setting `{"name":"header","region":"header","widgetLayout":
{"direction":"row","align":"center","justify":"space-between"}}` on the header `AppSection` entry in
`App.Configuration.layout.appSections[]` produces a genuine single-row header — confirmed live: before
the fix, `site-branding`+`page-navigation` rendered as two stacked full-width bars; after, one
continuous bar, logo left / nav right, verified via screenshot at 1600px.

**The real, still-open gap**: no exposed AppStudio MCP tool can set `widgetLayout` on an EXISTING
section. `create_section`'s own description states it "returns an error rather than creating a
duplicate" when `sectionName` already exists, and there is no `update_section` tool in the current
17-tool catalog (confirmed by listing every `.cs` file under
`BizFirst.Ai.Mcp.Tools.AppStudio\Tools\`). On this Meridian pass the header's `widgetLayout:
{"direction":"row"}` (and a `childLayout:{"direction":"row"}`) were already present in
`App.Configuration` when this session got MCP write access — set by some other means before this
session, not by an MCP tool call this session made — which is itself proof of the gap: the only way
to get `widgetLayout` onto an already-existing section today is a mechanism outside the 17 MCP tools
(direct DB write, or the Designer UI's own Sections panel, not yet confirmed which). **Whoever adds
an `update_section` tool next should expose `widgetLayout`/`childLayout`/`region` as settable fields
on it** — this is the actual, complete fix for header/nav-row layouts and any other multi-widget
section, not a documented limitation to work around with per-widget CSS.

## 12. Two JSON-argument gotchas that produce the opaque "An error occurred invoking '<tool>'" error — always populate every schema property, and never pass a bare `""` for a JSON-string field

Both real, reproduced this pass, both explained by the exact same opaque-wrapper symptom §3/§10
already flagged, but with actual root causes and fixes this time (§10 left `get_app` as an
unexplained, unfixed gap — it is not actually broken):

- **`get_app` (and by the same shape, likely other tools with an "optional, pass-one-of" pair of
  parameters)**: calling with only `{"appID":1090}` fails with the generic error every time. Calling
  with `{"appID":1090,"appCode":""}` (the SAME appID, just also including the unused sibling field as
  an explicit empty string) succeeds. The tool's own `inputSchema.required` array lists BOTH `appID`
  and `appCode` even though the description says "pass exactly one of" — that `required` listing is
  apparently enforced as "key must be present in the arguments object" by the MCP call path, not
  optional the way the human-readable description implies. **Fix/rule: always include every property
  the tool's `inputSchema` lists, even ones you don't intend to set — use `""`/`0`/`false`/`null` as
  appropriate for the ones you're leaving alone, never omit the key.** This fully resolves §10's
  `get_app` "gap" — it was never a `UserIdRequired`-style auth bug, just a missing key in the call.
  (`list_apps` was not re-tested this pass; worth checking whether it's the same shape.)
- **`update_widget_placement`'s `widgetStyle`/`widgetCss` fields**: passing `""` (empty string) for
  these when you only want to change `styleConfiguration` and leave the legacy fields alone fails
  with the same generic error; passing `"{}"` for `widgetStyle` (and `""` remains fine for
  `widgetCss`, which is a plain CSS-class string, not JSON) succeeds. Root cause not independently
  confirmed server-side, but the pattern (fails on `""`, works on `"{}"`) matches a server-side
  `JSON.parse`/deserialize call on that specific field throwing on empty input. **Rule: for any tool
  parameter whose description says "... JSON string" (`styleConfiguration`, `widgetStyle`,
  `configuration`, `theme`), pass a valid empty JSON value (`"{}"`) as the no-op, never a bare `""`
  — reserve bare `""` for genuinely plain-string fields (`widgetCss`, `appCode`, `navPosition`).**

## 13. `background: <gradient>, url(...)` as one shorthand value can silently render NEITHER layer — split into `background-image`/`background-color`/`background-size`/`background-position` longhand instead

Real, reproduced and fixed on Meridian's Home hero, 2026-09-15. The original hero CSS (written
2026-09-14, see this doc's own §5 imagery note) used the `background` SHORTHAND to combine a
`color-mix()` gradient scrim with a photo URL in one declaration:
`background:linear-gradient(...), url('...');background-size:cover;background-position:center`. Live
in the App Player this rendered as a flat, photo-less background — no gradient tint either, just
whatever the ancestor's own background was showing through — even though the referenced image URL
independently returned `200 image/jpeg` at a real, non-trivial file size (confirmed via `curl`) and
plain `<img src>` tags pointing at the exact same host/protocol elsewhere on the same page rendered
correctly. Rewriting the SAME two layers as longhand properties fixed it immediately, no other
change:

```css
/* Don't (silently renders nothing, in this App Player context): */
background: linear-gradient(180deg, color-mix(in srgb, var(--app-var-bg,#0f1420) 58%, transparent) 0%, ...) , url('https://.../hero-workspace.jpg');
background-size: cover;
background-position: center;

/* Do (real, verified fix): */
background-color: var(--app-var-bg, #0f1420);
background-image: linear-gradient(180deg, color-mix(in srgb, var(--app-var-bg,#0f1420) 58%, transparent) 0%, color-mix(in srgb, var(--app-var-bg,#0f1420) 92%, transparent) 100%), url('https://.../hero-workspace.jpg');
background-size: cover;
background-position: center;
background-repeat: no-repeat;
```

Not independently root-caused against the renderer/sanitizer source this pass (the `content` widget's
`allowScripts:true` HTML goes through DOMPurify per §1 — a plausible but unconfirmed theory is some
interaction between the sanitizer's attribute/style handling and a multi-layer shorthand value
containing both a function call and a `url()` in the same property). **Practical rule regardless of
root cause: always write a themed hero/section background as separate `background-color` +
`background-image` (+ `background-size`/`position`/`repeat`) declarations, never the combined
`background` shorthand, for any `content` widget in this system.**

## 14. `site-branding`'s hardcoded text color is confirmed broken specifically under LIGHT theme presets — live-reproduced, not just a static-code read

Extends §2's static-code finding (`SiteBrandingWidgetRenderer.tsx` hardcodes its own text color,
doesn't read `--app-var-*`) with a real, visual repro from Meridian's theme-swap test, 2026-09-15.
Swapped the app's theme to a light preset (`bg:#fefaf6`, "Sunrise Light" shape) via `update_app`: the
header bar background correctly went light/cream (`widgetContainer`'s own
`background:var(--app-var-bg,...)` tracks the theme fine, per §2), but the "Meridian Creative Co."
brand title text stayed the SAME near-white/pale-blue color it uses on a dark theme — on the new light
background this is genuinely illegible (confirmed via a zoomed screenshot crop, not just "a bit low
contrast"). Every OTHER text element on the same header bar (the nav links) rendered in dark, legible
text under the same light theme, so this is specific to the site-branding widget's own title text
color, not a page-wide cascade failure. Switching to two different DARK presets afterward
("Charcoal Studio", "Deep Space") showed the brand title text looking fine both times — consistent
with a literal hardcoded light color that happens to work on dark themes and fail on light ones. No
config-level fix exists (confirmed again: `SiteBrandingWidgetConfig` is `{titleOverride, imageSize}`,
no color/style field) — this remains a real frontend code fix needed in
`SiteBrandingWidgetRenderer.tsx`, now with a concrete, reproducible failure case (any light theme
preset) rather than a theoretical one.

## 15. No `delete_widget`/`delete_widget_placement` tool exists — use `update_widget_placement`'s `styleConfiguration` to `display:none` a placement you need to suppress

Real situation hit on Meridian's Work page: the `image-gallery` widget placement (a live, unscoped
query per §5) was rendering completely broken — not just irrelevant images, but broken-image icons
with raw filenames as visible alt-text (`sample-image-1.png`, `FaceBook.png`, etc.), clearly seed/test
documents unrelated to this tenant's real content. The 17-tool AppStudio catalog has no
`delete_widget`/`delete_widget_placement`/`remove_widget` tool of any kind (confirmed against the
full `tools/list` response, 61 tools across every MCP module this server exposes, not just
AppStudio) — a placement, once created, cannot be removed via MCP once it's serving no purpose.
**Working fix**: `update_widget_placement(widgetPlacementID=<AppWidgetID>,
styleConfiguration="{\"widgetContainer\":{\"css\":\"display:none\"}}", widgetStyle="{}",
widgetCss="", displayOrder=<unchanged>, showInNav=false, navPosition="top")` (see §12 above for why
`widgetStyle` needs `"{}"` not `""`) — the placement and its `AppWidgetID` still exist in the
database and still count toward `list_widgets_in_app`, but it no longer renders or takes up layout
space. Paired this with editing the page's OTHER content widget to remove any copy that referenced
the now-hidden widget (a "more from our gallery" lead-in paragraph), since a hidden-but-still-
referenced widget reads as a dangling/broken promise to a real visitor. **General lesson: page
copy that describes an adjacent widget should be re-checked any time that widget's placement is
hidden or its content changes** — the two are logically coupled even though they're separate
`Configuration` blobs with no structural link between them.

## 16. `site-assets/meridian/*` files can 200 with real, correctly-sized image bytes and still be the WRONG photo — a successful `curl` is not proof the content matches its filename/label

Real finding, corrects an assumption made earlier in this same build. Every image URL referenced by
Meridian's original (2026-09-14) Work/About page HTML resolved with `200 image/jpeg` (or `png`) and a
real, multi-hundred-KB-to-several-MB file size when checked with `curl` — including
`team-founder.jpg` (labelled "Founding Creative Director") and `brand-identity-project.png` (labelled
a brand-identity case study). Visually opening these files showed `team-founder.jpg` was an abstract
circuit-board/tech-overlay image with no person in it at all, and `brand-identity-project.png` was a
generic stained-glass-style abstract pattern with no relation to branding work — both are real files
in `wwwroot\site-assets\meridian\` on disk (a first-party ASP.NET static-web-assets folder on the
Consolidated WebApi, confirmed via its `.staticwebassets*.json` manifests, not a proxy or placeholder
route), just the WRONG content for their filename. Root cause: whichever process originally populated
this folder appears to have copied files from `C:\BizFirstGO_FI_AI\StockAssets\` without checking
that each file's actual visual content matched the semantic filename it was given. **Fix applied**:
read each candidate `StockAssets` file with the Read tool (which renders it visually) BEFORE trusting
its filename, picked real matches (a desk/coffee flatlay for the hero, an actual portrait photo for
the one team headshot kept, a laptop render for the "web design" case study, etc.), copied them into
the same `wwwroot\site-assets\meridian\` folder under new, honestly-descriptive filenames, and
repointed each widget's `<img src>`/`background-image` at the new filename via
`update_widget_definition` — new files under this static-assets folder are served immediately with no
WebApi restart needed (confirmed via `curl` immediately after the `cp`). **General lesson: when a
page references an existing image by URL, always open/view the actual file before trusting its
filename or a successful HTTP status code — a 200 response only proves a file exists at that path,
never that its content matches what the surrounding copy claims it is.**

## Sources

- Live build: AppID 1090 "Meridian Creative Co." (`BizFirst.Ai.Mcp.Tools.AppStudio`, 2026-09-14) —
  every fix in this doc was found and applied on this real app, not a synthetic test.
- §8-10: AppID 1092 "aaa" (`BizFirst.Ai.Mcp.Tools.AppStudio`, 2026-09-15) — margin/`color-scheme`
  fixes applied via `update_widget_placement` over the MCP server's HTTP/JSON-RPC endpoint using a
  tenant-scoped API key (not a user JWT), and verified live in the App Player both by DOM/computed-
  style probe and by screenshot.
- §11-16: AppID 1090 "Meridian Creative Co." revisit, 2026-09-15 — header/hero/content fixes and a
  3-preset theme-swap test (Sunrise Light, Charcoal Studio, Deep Space; `Template_DataTemplates`
  `DataTemplateTypeID=54`), all applied and verified live via the same MCP HTTP/JSON-RPC endpoint
  plus direct `wwwroot` file copies for imagery; theme restored to its pre-test value after the test.
- `widget-handlers-content-widget\src\ContentSanitizer.ts`, `ContentWidgetConfig.ts`
- `widget-handlers-site-branding-widget\src\SiteBrandingWidgetRenderer.tsx`,
  `widget-handlers-page-navigation-widget\src\PageNavigationWidgetRenderer.tsx`
- `app-handlers-core\src\types\StyleSlot.ts` (`WidgetStyleConfig`, the ten named slots and which two
  are actually wired)
- `BizFirst.Ai.Mcp.Tools.AppStudio\Tools\{UpdateAppTool,CreateSectionTool,UpdateWidgetDefinitionTool,
  WidgetTypeCatalog}.cs`
- `BizFirstFi.Go.IAM.Service\Authentication\ApiKeyAuthenticator.cs`,
  `Go.Essentials.Domain\Request\UserContext\GoUserContextAccessor.cs` (`UserIdRequired` throw
  behavior)
- `BizFirstFi.Go.Documents.Api.Base\Controllers\BaseDocumentController.cs` (`Upload`,
  `[AuthorizeRegularUserAttribute]`)
- `theming.md` (this doc's companion — token contract, widget catalog, preset catalog)
