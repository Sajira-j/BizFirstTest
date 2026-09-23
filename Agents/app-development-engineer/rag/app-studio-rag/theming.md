# App Studio Theming (`--app-var-*`) and the Widget Type Catalog

Tier 1 doc (see `00-overview.md`'s two-tier retrieval pattern). Load this whenever a request
involves: styling a `content` widget or any hand-written CSS that should track an app's look:
(the `--app-var-*` token contract), picking/configuring ANY of the 18 real widget types, or
sourcing a real image for a widget. Written 2026-09-14 (Task 89), source-grounded against real
code and a live query of `data-ocean-platform-prod` (`.\SQLEXPRESS`) — every JSON example below is
a real row, not invented. Companion to `styling-and-common-properties.md` (which carries a short
summary of the 12-original-token version of this contract — that summary is now stale on the
token count; this file is the authoritative, current version) and `content-widget-templates.md`
(the 4 template rows, cross-referenced below).

---

# Part 1 — The `--app-var-*` theme cascade

## 1.1 What it is and why it exists

Design source: `Documentation\WorkManagement\app-studio-theme\{design.md,plan.md}`. Before this
existed, every `content` widget's `<style>` block hardcoded literal hex/px values — changing a
brand color meant hand-editing every widget. A prior idea to reuse generic names (`--bg-dark`,
`--text-primary`) was rejected because THREE different, already-live, mutually-colliding schemes
already used those exact names elsewhere in the codebase (`shared-styles/design-tokens.css`'s
cross-product tokens, the App Player host page's own `:root` values, and `AppPlayer.tsx`'s inline
`--color-bg-primary`) — see `design.md` §1. `--app-var-*` is a deliberately namespaced, DOM-scoped
custom-property set that never touches `:root` and never collides with any of those.

**One resolved layer, most-specific-wins, browser-native cascade — no JS-side merging.** Three
nested DOM scopes, outermost to innermost, each `<style>` tag setting ONLY the tokens that layer
explicitly overrides; anything unset simply inherits from the parent scope via ordinary CSS custom
property inheritance:

```
[data-theme-scope="tenant"]        -- tenant's own AppStudioTheme.Tokens, fetched as pre-rendered CSS
  [data-theme-scope="app"][data-app-id="{id}"]   -- the App's own Configuration.theme
    [data-page-id="{id}"]                         -- the active Page's own ThemeConfiguration
```

All three layers, plus the built-in defaults layer, are implemented once in
`app-handlers-generic\src\hooks\themeInjector.ts` and reused by every real consumer found in this
codebase (`AppPlayer.tsx`, WorkDesk, the Designer's live-preview editors) — **never reimplement
this CSS-generation/sanitization logic; import and call the existing functions.**

Confirmed **fully built and live** as of today (not just designed): engine plumbing
(`app-handlers-generic\DevelopmentHistoryLog.md`, 2026-09-11 entry), the tenant backend
(`BaseAppStudioThemeAssetsController.cs` / `AppStudioTenantThemeService.cs`,
`GET /api/assets/css/app-studio-theme.css`), the 90+-row preset catalog (§1.7 below), and the
Theme Editor UI (`AppConfigScreen.tsx:9,33,175` wires a live `AppThemeTab` into the Designer's
"Theme" tab) — all four plan.md phases have shipped.

## 1.2 The exact CSS pattern — always a real fallback, never a bare `var()`

```css
/* property: var(--app-var-<token>, <original-hardcoded-fallback>); */
background: var(--app-var-color-primary, #4a90d9);
```

The fallback is what renders (a) before any theme is injected, (b) in any DOM context that never
gets a `[data-theme-scope]`/`[data-app-id]` ancestor at all (an editor preview, a tool surface
outside App Studio, this doc's own JSON examples read outside a browser), and (c) is what the
literal color would have been if you'd hardcoded it — never a placeholder guess. A bare
`var(--app-var-x)` with no second argument resolves to the CSS-wide initial value (nothing) the
instant it's outside every scope, which is silently blank, not an error — see the real bug in
§1.6.

## 1.3 The full canonical token list — 19 tokens

Source of truth: `app-handlers-core\src\types\ThemeTokens.ts:33-79` (`APP_THEME_TOKENS`). The
first 12 are the original set; the last 7 were added the same day as a Wix/Shopify-SDK-aligned
follow-up (each independently overridable — e.g. overriding `color-primary` does NOT move
`button-bg`, by design).

| Token (`key` / `--app-var-{key}`) | Typical use | Default value |
|---|---|---|
| `color-primary` | brand accent — buttons, links, highlighted text, active states | `#4a90d9` |
| `color-secondary` | a second accent, used more sparingly | `#e8a33d` |
| `bg` | the app's base/darkest background | `#1a1a2e` |
| `bg-panel` | a one-step-lighter panel/card/section background | `#242438` |
| `text` | primary/heading text color | `#f0f0f5` |
| `text-muted` | secondary/caption/label text color | `#9a9ab0` |
| `border` | hairline borders, dividers | `#33334d` |
| `font-family` | body font stack | `system-ui, -apple-system, "Segoe UI", sans-serif` |
| `font-heading` | heading font stack | `system-ui, -apple-system, "Segoe UI", sans-serif` |
| `radius` | corner radius for buttons/cards/inputs | `8px` |
| `shadow` | box-shadow for elevated elements | `0 2px 8px rgba(0,0,0,0.24)` |
| `space-unit` | base spacing unit for gap/padding rhythm | `8px` |
| `color-primary-hover` | hover state of `color-primary` | `#5da0e8` |
| `link` | link color, independent of `color-primary` (Wix-style separation) | `#4a90d9` |
| `text-inverse` | text color for content ON a colored/dark button or accent | `#ffffff` |
| `success` | success/positive state color | `#3ecf7e` |
| `error` | error/negative state color | `#e5484d` |
| `button-bg` | button background — independently overridable from `color-primary` | `#4a90d9` |
| `button-text` | button text — independently overridable from `text-inverse` | `#ffffff` |

**Do not invent a 20th token or reuse a shared-tool-chrome variable** (`--bg-dark`,
`--text-primary`, `--color-bg-primary` — the exact names `design.md` §1 shows already collide). A
widget needing something this list doesn't cover falls back to a literal for that one property.

**Buttons specifically should use `button-bg`/`button-text`**, not `color-primary`/`text-inverse`
directly — real example, `AIExt_Widgets` 848 (`.tpl-hc-btn.primary{background:var(--app-var-button-bg,#4a90d9);color:var(--app-var-button-text,#ffffff)}`).

**Alpha-blended/translucent variant of a token** (a glow, a subtle border) — the token holds a full
CSS value (hex, keyword, gradient stop), not bare R,G,B components, so `rgba(var(--app-var-color-primary), 0.2)`
does NOT work. Use `color-mix()` instead — real example, widget 792 (Timeline template):
`box-shadow:0 0 0 4px color-mix(in srgb, var(--app-var-color-primary,#4a90d9) 25%, transparent)`.

## 1.4 How a NEW consumer integrates with the cascade

Two real, different integration shapes exist in this codebase today — pick the one that matches
your consumer:

**Shape A — full 3-layer nested cascade (what `AppPlayer.tsx` itself does).** Use this when your
consumer already has real, separate tenant/app/page-scoped theme data to inject as three distinct
layers:
1. Call `ensureDefaultThemeInjected()` once (idempotent) — injects the built-in defaults scoped to
   `[data-theme-scope="app"]`.
2. Stamp `data-theme-scope="app"` and `data-app-id={appID}` on your root wrapper element.
3. Call `injectAppThemeOverride(appID, app.configuration.theme)`.
4. If you also have a page-scoped override, wrap the page content in a descendant node carrying
   `data-page-id={pageID}` and call `injectPageThemeOverride(appID, pageID, page.themeConfiguration)`.
5. If you also need the tenant layer, fetch `GET /api/assets/css/app-studio-theme.css?tenantID={id}`
   and call `injectTenantThemeCss(cssText)` — this one takes pre-rendered CSS text, not a token
   map, since it's the only layer requiring a network fetch.

**Shape B — single collapsed "resolve-then-inject-once" layer (what WorkDesk does — real,
working code to copy).** Use this when your consumer is NOT itself scoped to one specific App
Studio App the way `AppPlayer.tsx` is (no natural per-page or per-tenant DOM boundary of its own).
WorkDesk's exact, real implementation
(`hil\app\src\hooks\useAppStudioThemeInjection.ts`,
`hil\app\src\config\workDeskThemeConfig.ts`,
`hil\app\src\services\workDeskTenantSettingsClient.ts`):

1. **Resolve which AppID supplies the theme** via a three-tier fallback chain, highest priority
   first (`workDeskThemeConfig.ts:15-28`):
   1. A per-tenant `WorkDesk.AppID` row in the generic `IAM_TenantSettings` key-value store (read
      via `workDeskTenantSettingsClient.ts`'s `getWorkDeskAppID()`, `SettingKey = 'WorkDesk.AppID'`).
   2. A deployment env var (`VITE_WORKDESK_DEFAULT_THEME_APP_ID`).
   3. A hardcoded fallback (`WORKDESK_DEFAULT_THEME_APP_ID_FALLBACK = 1`).
2. **Fetch that resolved App's own theme.** If the app doesn't exist, or exists with no
   `Configuration.theme` of its own, fall through to the tenant's own theme token map instead
   (`APP_STUDIO_TENANT_THEME_TOKENS_SETTING_KEY = 'AppStudioTheme.Tokens'`, same generic
   `IAM_TenantSettings` store — read raw JSON, no dedicated endpoint needed since
   `AuthorizeRegularUserAttribute` already permits it).
3. **Inject the single resolved result once** via `injectAppThemeOverride(resolvedAppID, effectiveTheme)`
   — no separate tenant-scoped DOM layer competing with the app layer. Binoy's own framing of this
   rule (`useAppStudioThemeInjection.ts:41`): *"Tenant theme is not directly applied. We try to
   apply app theme. if app theme is empty, we use tenant[']s [theme] as a default mechanism."*
4. Stamp `data-theme-scope="app"` + `data-app-id={themeAppID}` on your root wrapper — WorkDesk does
   this on `AuthenticatedApp`'s outermost div (`App.tsx:49`), matching `AppPlayer.tsx`'s own root
   attribute shape exactly so the SAME globally-injected `<style>` tags apply without any
   consumer-specific selector.

**Either shape — the wrapper element/attribute contract is non-negotiable**: without a real DOM
ancestor carrying `data-theme-scope="app"` (Shape A) or the same attribute (Shape B) plus
`data-app-id`, `var(--app-var-*)` resolves to nothing anywhere inside it, defaults included — see
the real bug in §1.6.

## 1.5 Real before/after CSS — hand-written consumer stylesheets

`bizfirst-common\chatdesk-chat-window\src\styles\chatWindow.css:37,45,53,74-77,151,153-154,228-230,242,284`
(ChatDesk's Chat Panel widget target) — every rule that used to be a bare literal now degrades
through the app's OWN pre-existing custom-property chain as a second fallback, so a consumer with
no App Studio scope at all still renders exactly as before:

```css
/* Real, current chatWindow.css */
.chatdesk               { background: var(--app-var-bg, var(--bg-primary, var(--color-bg))); }
.chatdesk-header         { border-bottom: 1px solid var(--app-var-border, var(--border, var(--color-border))); }
.chatdesk-send-button    { background: var(--app-var-button-bg, var(--color-primary, var(--accent-1)));
                            color: var(--app-var-button-text, #fff); }
.chatdesk-devinfo-link   { color: var(--app-var-link, var(--color-primary, var(--accent-1))); }
```

`bizfirst-common\hil-ui-chat-window\src\styles\hil-chat.css:144-152` — note the doc comment there
explaining WHY only two rules in that file get the `--app-var-*` tier: most of the file's
`--hil-chat-*` variables are computed at `:root` (can't see a scoped `[data-app-id]` ancestor
deeper in the tree), but the bubble-content rules apply directly on real descendant elements of
that scope, so only those two get the extra fallback tier:

```css
.hil-chat-bubble--user .hil-chat-bubble__content   { background: var(--app-var-color-primary, var(--hil-chat-bubble-user-bg));
                                                       color: var(--app-var-text-inverse, var(--hil-chat-bubble-user-text)); }
.hil-chat-bubble--system .hil-chat-bubble__content { background: var(--app-var-bg-panel, var(--hil-chat-bubble-system-bg));
                                                       color: var(--app-var-text, var(--hil-chat-bubble-system-text)); }
```

WorkDesk's own CSS files (`hil\app\src\components\hil\hil.css:28-31`,
`...\features\inbox\inbox-item.css:60-62,117,120,130`,
`...\features\task\task-header.css:8-10,153-159`) show the same pattern applied directly (no
intermediate `--hil-*` tier, since these are WorkDesk-local, not shared-package, styles):

```css
.hil-overlay__dialog        { background: var(--app-var-bg-panel, rgba(255,255,255,0.98));
                               border-radius: var(--app-var-radius, var(--border-radius-lg, 14px));
                               border: 1px solid var(--app-var-border, #e2e8f0);
                               box-shadow: var(--app-var-shadow, 0 20px 50px rgba(15,23,42,0.22), 0 0 0 1px rgba(34,197,94,0.06)); }
.hil-inbox-item--unread      { border-left: 3px solid var(--app-var-color-primary, #3b82f6) !important; }
.hil-task-header__action-btn { background: var(--app-var-bg-panel, var(--bg-secondary, #f9fafb));
                                border: 1px solid var(--app-var-border, var(--border-default, #e5e7eb));
                                border-radius: var(--app-var-radius, var(--border-radius-base, 6px));
                                color: var(--app-var-text, var(--text-primary)); }
```

Real before/after for a `content` widget's own inline `<style>` — `AIExt_Widgets` WidgetID 848
("Hero — Centered", one of 30 new content templates seeded today, 2026-09-14):

```css
/* Don't (a literal, pre-theming version of this exact widget): */
.tpl-hc { padding: 96px 32px; background: #1a1a2e; font-family: system-ui, -apple-system, "Segoe UI", sans-serif; }
.tpl-hc-btn.primary { background: #4a90d9; color: #ffffff; }

/* Do (the real, current row): */
.tpl-hc { padding: calc(var(--app-var-space-unit,8px)*12) calc(var(--app-var-space-unit,8px)*4);
          background: var(--app-var-bg,#1a1a2e);
          font-family: var(--app-var-font-family,system-ui, -apple-system, "Segoe UI", sans-serif); }
.tpl-hc-btn.primary { background: var(--app-var-button-bg,#4a90d9); color: var(--app-var-button-text,#ffffff); }
```

## 1.6 Common mistakes (found and fixed today, real bugs — not hypothetical)

1. **Forgetting the fallback entirely.** `var(--app-var-color-primary)` with no second argument
   resolves to nothing the instant the element isn't inside a scoped ancestor — silent, not an
   error. Always pair every `--app-var-*` reference with the literal it would have been.

2. **An editor/preview surface not wrapped in the theme-scope attributes — real, live-reproduced
   bug, fixed 2026-09-11/12.** Root-caused in
   `app-studio-designer-components-react\DevelopmentHistoryLog.md:156-192`: the Content Widget
   Style Editor's Tiptap-based `ContentVisualEditor.tsx` preview rendered a widget whose color
   declaration was `color: var(--app-var-color-primary)` (a theme token, not a literal) as
   invisible/wrong-colored text. **This was never a Tiptap or CSS-parsing bug** — the round-trip
   through `ContentStyleModel.parseStyleBlock`/`regenerateStyleBlock` was byte-identical. The real
   cause: none of `AppPlayer.tsx`'s three theme-cascade scope roots were ancestors of the Widget
   Details overlay this editor renders inside — a separate part of the Designer's own tree, never
   a descendant of the live-preview `AppPlayer` instance — so `var(--app-var-color-primary)`
   resolved to `''` inside that preview DOM. **Fix, and the pattern to copy for any new
   editor/preview surface**: `ContentVisualEditor.tsx` gained optional `appID`/`pageID` props; when
   supplied, the preview's outer wrapper carries `data-theme-scope="app" data-app-id={appID}` with
   `data-page-id={pageID}` nested beneath — reproducing the exact DOM shape `AppPlayer.tsx` uses,
   so the SAME already-injected `<style>` tags (they live in `<head>`, document-global) cascade
   into the editor's DOM correctly. **Any new preview/editor surface for themed content must do the
   same** — rendering theme-token CSS without these wrapper attributes will silently show nothing
   or the wrong color, and it will look like a parsing bug when it is actually a missing-scope bug.

3. **Confusing app-layer vs. page-layer theme overrides — real, live-diagnosed case.** Same
   incident (`DevelopmentHistoryLog.md:170-173`): the widget in question rendered orange
   (`#ffaa00`), not the tenant-wide default blue (`#4a90d9`). Reading the injected stylesheet
   directly showed AppID 1069 has **no app-level override at all** — the color came from that
   app's Home page's **own page-level override**
   (`[data-app-id="1069"] [data-page-id="27"]`), one layer more specific than the app layer. Both
   layers are real, independently settable, and the more specific one silently wins with zero
   visual cue which layer supplied a given value — when debugging "why is this the wrong color,"
   always check BOTH `AIExt_Apps.Configuration.theme` (app scope) and the active page's own
   `ThemeConfiguration` (page scope, one level more specific) before assuming the app-level value
   is the one in effect.

## 1.7 The `AppStudioTheme` preset catalog (`DataTemplateTypeID = 54`)

For anyone building a theme picker/selector UI, or wanting a ready-made coherent token bundle
instead of hand-picking 19 values: `Template_DataTemplates` rows with `DataTemplateTypeID = 54`
are real, `IsGlobal = 1`, `Published = 1` preset bundles — a tenant/app/page picks one and its full
`ContentData` JSON (the complete `--app-var-*` key/value map, using the bare keys, not the
`--app-var-` prefixed names) is COPIED into that scope's own storage — copy-on-select, frozen
thereafter (design.md §5): a later edit to the preset itself does not retroactively repaint
anything that already copied it.

Confirmed live (query `SELECT DataTemplateID, TemplateName, ContentData FROM Template_DataTemplates
WHERE DataTemplateTypeID = 54` against `data-ocean-platform-prod`): **90+ real rows**, e.g.:

```json
// DataTemplateID 91020393 — "Ocean Breeze"
{"color-primary":"#0ea5e9","color-secondary":"#14b8a6","bg":"#f0f9ff","bg-panel":"#ffffff",
 "text":"#0c2333","text-muted":"#5b7690","border":"#d6e9f5",
 "font-family":"\"Inter\", \"Helvetica Neue\", Arial, sans-serif", "radius":"10px", ... }

// DataTemplateID 91020394 — "Charcoal Studio"
{"color-primary":"#6366f1","color-secondary":"#a78bfa","bg":"#111114","bg-panel":"#1b1b21",
 "text":"#f2f2f7","text-muted":"#93939f","border":"#2a2a33", "radius":"6px", ... }
```

Applying a preset = write its `ContentData` into the target's own storage (`AIExt_Apps.Configuration.theme`
for app scope, `AIExt_AppPages.ThemeConfiguration` for page scope, or the tenant's
`AppStudioTheme.Tokens` `IAM_TenantSettings` row) — never a live pointer back to the preset row.
This is exactly what the live `AppThemeTab` (`AppConfigScreen.tsx`) does today.

---

# Part 2 — Widget type catalog (all 18 real `WidgetType` values)

Confirmed authoritative list: `WidgetTypeCatalog.cs:25-45` (`RealWidgetTypes`,
`BizFirst.Ai.Mcp.Tools.AppStudio\Tools\WidgetTypeCatalog.cs`) — its own doc comment notes
`00-overview.md`'s "17" header text is a stray typo; the table there, and this catalog, both list
18. Every example JSON below is a real, live `AIExt_Widgets.Configuration` row from
`data-ocean-platform-prod`, queried directly — not fabricated.

## 2.1 Known current limitation — read before trusting "valid" as "correct"

`WidgetTypeCatalog.cs`'s own doc comment (`:9-21`): structural/presence validation (does the JSON
have the required key, non-empty) exists for only **9 of the 18 types** — `content`, `image`,
`video`, `audio`, `pdf`, `form`, `workflow-template`, `workflow-template-category`, `chat-panel`.
The other **9 — `page-navigation`, `hil-inbox`, `signin`, `notifications`, `site-branding`, and all
4 gallery types — get NO structural validation at all (pass-through).** Separately, and more
importantly for the 4 cross-domain reference types (`form`/`workflow-template`/
`workflow-template-category`/`chat-panel`): only field PRESENCE is checked, never that the
referenced Atlas Forms/Flow Studio entity actually exists or belongs to the same tenant — a
`create_widget` call can reference an ID that doesn't exist, or belongs to another tenant, and this
module accepts it. Real, open follow-up work, not a solved problem — don't assume a successful
`create_widget`/`update_widget_placement` call proves the config is correct; per `widgets\content.md`'s
own hard-won lesson, a validator agreeing is not the same as the real renderer showing anything.

## 2.2 Theme-cascade participation — the honest picture

Grepping every `widget-handlers-*` package for `app-var` found matches in exactly **one** place:
none. **No widget type's own handler/renderer CSS references `--app-var-*` today.** The 18
structural widget types render through fixed handler chrome with no theme-token wiring at all.
**The only widget type with real `--app-var-*` participation is `content`**, and only because its
`allowScripts: true` escape hatch lets an author write arbitrary `<style>` CSS by hand (see
`styling-and-common-properties.md`'s "structured style system cannot do hover/animation" section
for why this is the sanctioned way to get real theming into a widget at all — the structured
`style`/`css` slots on every OTHER widget type are single-element, non-selector, and don't carry
theme tokens either). Every `content` row seeded with real theming (the 4 Content Widget Templates,
792-795, and the 30-row 2026-09-14 batch, 848-877) does it by hand-writing `var(--app-var-*, fallback)`
inside its own `content`/`allowScripts` HTML string — there is no separate "theme" config field on
`ContentWidgetConfig` itself. Per-type participation is noted individually below; where it says
"none," that is a confirmed absence, not an oversight in this doc.

## 2.3 Per-type reference

### `content` — Content Widget
**For**: static HTML, Markdown, or plain-text content — the only widget type with a real styling
escape hatch. **Config** (`ContentWidgetConfig.ts`, full detail in `widgets\content.md`):
`content` (string, required), `format` (`'html'|'markdown'|'text'`, required), `allowScripts`
(boolean, default `false`). **Real example** (WidgetID 792, minus the full CSS block — see §1.5/§1.7
for full real CSS): `{"format":"html","allowScripts":true,"content":"<div class=\"tpl-timeline\">…"}`.
**Theming**: full participation — see Part 1 throughout. **Validated**: yes (presence of `content`
non-empty + `format` in the 3-value enum).

### `form` — Form Widget
**For**: create/edit/view/list records from an Atlas Forms form — the generic-CRUD building block.
**Config**: `formId` (number, required), `mode` (string, e.g. `'edit'`/`'list'`). **Real example**
(WidgetID 717, "Documents Search"): `{"formId":30500,"mode":"edit"}`. **Theming**: none — rendering
is delegated entirely to Atlas Forms' own `FormRenderer`, outside this scan's `app-var` grep.
**Validated**: yes (`formId` presence) — but per §2.1, existence/tenant-ownership of `formId` 30500
itself is NOT checked.

### `image` — Image
**For**: a single photo — URL, alt text, caption, object-fit, optional link. **Config**:
`imageUrl` (required), `altText`, `caption`, `objectFit`, `linkUrl`, `presentationStyle`. **Real
example** (WidgetID 725, "Home Portrait"): `{"imageUrl":"https://upload.wikimedia.org/wikipedia/commons/0/04/Michael_Jackson_1984.jpg","altText":"Michael Jackson, White House, May 14, 1984","caption":"Michael Jackson at the White House, May 14, 1984 (White House Photo Office - public domain)","objectFit":"contain","linkUrl":"","presentationStyle":"polaroid"}`.
**Theming**: none. **Validated**: yes (`imageUrl` presence only).

### `video` — Video
**For**: a single video player. **Config**: `videoUrl` (required), `posterUrl`, `caption`,
`autoplay`, `loop`, `controls`. **Real example** (WidgetID 745): `{"videoUrl":"https://www.w3schools.com/html/mov_bbb.mp4","posterUrl":"","caption":"","autoplay":false,"loop":false,"controls":true}`.
**Theming**: none. **Validated**: yes (`videoUrl` presence only).

### `audio` — Audio
**For**: a single audio player. **Config**: `audioUrl` (required), `title`, `caption`, `autoplay`,
`loop`. **Real example** (WidgetID 746): `{"audioUrl":"https://www.w3schools.com/html/horse.mp3","title":"","caption":"","autoplay":false,"loop":false}`.
**Theming**: none. **Validated**: yes (`audioUrl` presence only).

### `pdf` — PDF
**For**: a single PDF — link or inline embed. **Config**: `pdfUrl` (required), `title`,
`displayMode` (`'link'`|embed). **Real example** (WidgetID 747): `{"pdfUrl":"https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf","title":"","displayMode":"link"}`.
**Theming**: none. **Validated**: yes (`pdfUrl` presence only).

### `image-gallery` / `video-gallery` / `audio-gallery` / `pdf-gallery`
**For**: a LIVE, filterable query against public assets (`IsPublicAsset = true` only — a hard
backend boundary, not a UI filter) — never a fixed chosen list; don't conflate with the single-media
types above. **Config** (shared shape, `mediaCategory` distinguishes them): `title`,
`documentTypeID`, `documentCategoryID`, `mediaCategory` (`'Image'|'Video'|'Audio'|'Pdf'`), `theme`
(a named visual theme — "classic-grid" etc.), `automation`, `columns`, `maxItems`, and (pdf only)
`displayMode`. **Real examples**:
- image-gallery (WidgetID 749): `{"title":"","documentTypeID":null,"documentCategoryID":null,"mediaCategory":"Image","theme":"classic-grid","automation":"off","columns":3,"maxItems":24}`
- video-gallery (WidgetID 750): same shape, `"mediaCategory":"Video"`, `"columns":3`.
- audio-gallery (WidgetID 751): same shape, `"mediaCategory":"Audio"`, `"columns":2`.
- pdf-gallery (WidgetID 752): same shape plus `"displayMode":"link"`, `"mediaCategory":"Pdf"`.

**Theming**: none directly in Configuration — visual variety comes from the `theme` field's own
named presets (5 for image, 3 for video, per `00-overview.md`), a SEPARATE mechanism from
`--app-var-*`, not cross-wired to it. **Validated**: NO structural validation (pass-through, per
§2.1) — a typo in `mediaCategory` or an out-of-range `columns` is currently accepted silently.

### `workflow-template` — Workflow Agent
**For**: one AI agent/execution template, with Execute or Chat Now. **Config**:
`executionTemplateID` (number, required). **Real example** (WidgetID 754): `{"executionTemplateID":2}`.
**Theming**: none. **Validated**: yes (presence only) — existence/tenant-ownership of
`executionTemplateID` NOT checked (§2.1).

### `workflow-template-category` — Workflow Category
**For**: a grid of every agent in one execution-template category. **Config**:
`executionTemplateCategoryID` (number, required). **Real example** (WidgetID 755):
`{"executionTemplateCategoryID":47}`. **Theming**: none. **Validated**: yes (presence only, same
existence caveat as above).

### `chat-panel` — Chat Panel
**For**: an embedded chat window for triggering/conversing with one process — this is the ChatDesk
integration point (§1.5's `chatWindow.css` is what actually renders inside it). **Config**:
`processID` (number, required), `enableConversationList` (boolean). **Real examples**: WidgetID 267
`{"processID":1,"enableConversationList":true}`; WidgetID 756 `{"processID":1073,"enableConversationList":true}`.
**Theming**: indirect but real — the widget's OWN Configuration carries no theme fields, but its
rendered chrome (`chatWindow.css`) is the `--app-var-*`-aware stylesheet documented in §1.5, so this
widget type visually themes correctly even though `WidgetTypeCatalog.cs` never touches theming.
**Validated**: yes (`processID` presence only), existence caveat as above.

### `page-navigation` — Page Navigation
**For**: a placeable menu of app pages, vertical or horizontal, nested-page aware. **Config**:
`orientation` (`'horizontal'`|`'vertical'`). **Real example** (WidgetID 162, "Sample: Top
Navigation Bar"): `{"orientation":"horizontal"}`. **Theming**: none. **Validated**: NO structural
validation (pass-through, §2.1).

### `hil-inbox` — HIL Inbox
**For**: a human-in-the-loop actionable inbox — approvals, forms, tasks. **Config**: none (auth/
tenant scoping only). **Real example** (WidgetID 163, "Sample: HIL Inbox"): `{}`. **Theming**:
none in the widget's own Configuration — note WorkDesk's OWN inbox chrome
(`inbox-item.css`, §1.5) is themed, but that is a separate app (WorkDesk), not this App Studio
widget type rendering inside an App Player app. **Validated**: NO structural validation.

### `signin` — Sign In / Sign Out
**For**: sign-in link when signed out; user menu with sign-out when signed in. **Config**: none
required (optional fields exist per `00-overview.md` but this scan found no seeded non-empty
example). **Real example** (WidgetID 164, "Sample: Sign In"): `{}`. **Theming**: none. **Validated**:
NO structural validation.

### `notifications` — Notifications
**For**: a notifications bell with unread count and dropdown list. **Config**: optional, no seeded
non-empty example found. **Real example** (WidgetID 165, "Sample: Notifications"): `{}`.
**Theming**: none. **Validated**: NO structural validation.

### `site-branding` — Site Branding
**For**: the App's own logo and name, side by side. **Config**: optional, no seeded non-empty
example found. **Real example** (WidgetID 166, "Sample: Site Branding"): `{}`. **Theming**: none.
**Validated**: NO structural validation.

---

# Part 3 — Local stock image library (`StockAssets`)

`C:\BizFirstGO_FI_AI\StockAssets\` holds **163 real image files** (jpg/png mix) as of today — e.g.
`Abstract-Background.jpg`, `Businessman.jpg`, `Avatar-Client.png`, `Background-WaterColor.jpg`,
`LaptopWithaCupofTea-office.jpg`. This is a plain folder on disk, curated as a stock-photo source
for App Studio content authors/agents to pick from when building `image`, hero-style `content`
templates, or gallery widgets — use a real photo from here instead of a placeholder/broken URL when
a request calls for imagery and no specific asset is supplied.

**How to browse it**: `C:\BizFirstGO_FI_AI\StockAssets\preview.html` — a static gallery page (open
directly in a browser) showing every file as a thumbnail with its filename as a caption, plus a
filename filter box. Use this to pick a specific filename before referencing it, rather than
guessing at what's in the folder.

**How to actually reference one in a widget** — this folder is NOT itself a served web endpoint
(confirmed: `preview.html`'s own `<img src="./Filename.jpg">` references only work because the page
is opened as a local file next to the images; there is no `/StockAssets/...` HTTP route wired into
App Studio's runtime). Every real seeded `imageUrl`/`videoUrl`/etc. example in Part 2 is a public
HTTPS URL (e.g. Wikipedia, w3schools' test-asset host) — the `image`/`video`/`audio`/`pdf` widget
types and all 4 gallery types only ever resolve a real, already-hosted URL (galleries additionally
require the asset be flagged `IsPublicAsset = true` in App Studio's own media library, per
`00-overview.md`'s security note). To use a `StockAssets` file in a real widget: upload it through
App Studio's own asset/media manager first (which gives it a real hosted URL and, if needed, the
`IsPublicAsset` flag), then use that resulting URL as the widget's `imageUrl`/`videoUrl`/etc. —
`preview.html` is a picking/browsing aid for choosing WHICH file to upload, not a shortcut around
the upload step.

---

## Sources

- `Documentation\WorkManagement\app-studio-theme\design.md`, `plan.md`
- `app-handlers-core\src\types\ThemeTokens.ts`, `AppConfiguration.ts`
- `app-handlers-generic\src\hooks\themeInjector.ts`, `AppPlayer.tsx`, `DevelopmentHistoryLog.md`
  (2026-09-11 and 2026-09-11/12 entries)
- `app-studio-designer-components-react\DevelopmentHistoryLog.md` (`ContentVisualEditor.tsx`
  theme-scope bug, `AppThemeTab`/`AppConfigScreen.tsx` Theme Editor UI)
- `hil\app\src\hooks\useAppStudioThemeInjection.ts`, `src\config\workDeskThemeConfig.ts`,
  `src\services\workDeskTenantSettingsClient.ts`, `src\App.tsx`,
  `src\components\hil\hil.css`, `src\features\inbox\inbox-item.css`, `src\features\task\task-header.css`
- `bizfirst-common\chatdesk-chat-window\src\styles\chatWindow.css`,
  `bizfirst-common\hil-ui-chat-window\src\styles\hil-chat.css`
- `BizFirst.Ai.Mcp.Tools.AppStudio\Tools\WidgetTypeCatalog.cs`
- `BizFirstFi.Go.IAM.Api.Base\Controllers\BaseAppStudioThemeAssetsController.cs`,
  `BizFirstFi.Go.IAM.Service\Services\AppStudioTheme\AppStudioTenantThemeService.cs`
- `data-ocean-platform-prod` (`.\SQLEXPRESS`), live queries against `AIExt_Widgets` and
  `Template_DataTemplates` (`DataTemplateTypeID = 54`), 2026-09-14
- `C:\BizFirstGO_FI_AI\StockAssets\` folder listing and `preview.html`
