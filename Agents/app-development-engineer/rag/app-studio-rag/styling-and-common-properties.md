# Common Properties & the Structured Style System

Migrated and current-as-of-2026-09-04 from the superseded `rag/_archive/app-studio-rag-v1-2026-08-30`
set (originally scanned 2026-08-30 from `app-handlers-core/src/types/{AppSection,StyleSlot}.ts` and
`atlas-forms/packages/designer-components-react/src/components/StyleBuilderPanel/types.ts`) — spot-
check against current source if picking this up much later, the same discipline this whole doc set
follows elsewhere.

## Section-level shared properties (`AppSection`)

| Property | Type | Default when omitted | Notes |
|---|---|---|---|
| `name` | string | — required | join key `AppWidgetRecord.sectionName` points at; renaming a section must cascade-update every widget's `sectionName`. |
| `appSections` | `AppSection[]?` | none — flat leaf | **recursive** — a section can contain child sections. |
| `region` | `'header'\|'left'\|'right'\|'main'\|'footer'?` | implicit `'main'`, legacy flex/flow container | setting `region` on even ONE section in a layout switches the whole renderer to a named-region CSS Grid. |
| `widgetLayout` | `AppSectionWidgetLayout?` | plain stacked block children, no flex container | `{direction?:'row'\|'column', wrap?, gap?, align?, justify?}` |
| `isPrimaryContentSection` | boolean? | `false` | marks the ONE section whose widgets can carry a page-scoping `appPageID`. |
| `sectionContainer` / `sectionBackground` | `StyleSlotValue?` | unstyled | Level 2 style slots — see below. |
| `hiddenOn` | `('mobile'\|'tablet'\|'desktop')[]?` | visible everywhere | |

## The `IWidgetHandler` contract every widget type implements

```ts
interface IWidgetHandler {
  readonly widgetType: string;
  load(widget: WidgetRecord): Promise<void>;
  render(ctx: WidgetRenderContext): WidgetRenderResult;   // pure data resolution, no DOM/React access
  unload(widgetId: string): Promise<void>;
  readonly Component?: ComponentType<WidgetHandlerRenderProps>;
}
```

`render()` returns a small discriminated-union `WidgetRenderResult` per type — never the widget's own
raw `configuration` directly. Dispatch is a plain `registry.get(widgetType)` lookup.

## Minimal valid widget placement

```json
{ "sectionName": "hero", "displayOrder": 1, "isDefault": false, "showInNav": false, "navPosition": null, "routable": false, "configuration": null }
```

`configuration: null` inherits the shared Widget's own base config unchanged — only set a
placement-level override when this specific instance genuinely needs to differ.

## Structured Style Builder System

**Always prefer the structured `style` object over the `css` escape hatch** — typed, enumerable,
never rejected by the sanitizer.

```ts
interface StyleSlotValue { css?: string; style?: StyleProperties; }
```

| Level | Lives on | Field names |
|---|---|---|
| 1 — whole App | `App.styleConfiguration` | `siteContainer`, `siteBackground` (edit-side only, not yet rendered) |
| 2 — a Section | `AppSection` | `sectionContainer`, `sectionBackground` |
| 3 — a Widget placement | `AppWidgetRecord.styleConfiguration` | `widgetContainer`/`widgetBackground` (wired), + 8 more named-but-unwired slots (`widgetHeader`/`widgetFooter`/`widgetBody`/`widgetContent`/`widgetImage`/`widgetText`/`widgetIcon`/`widgetTitle`) |

`style: StyleProperties` (~90 optional camelCase properties, cast straight to `React.CSSProperties`,
no shorthand): color/background, typography, box-model (all numbers, per-side, no `padding`
shorthand), sizing (strings, e.g. `"480px"`/`"100%"`), overflow, border (numbers, per-side), outline/
shadow/effects, layout (`display`/`flexDirection`/etc.).

**`css: string`** — raw CSS declarations only (`"color: red; padding: 8px;"`), never selectors/
rules. **Hard sanitizer rule**: any `{`, `}`, or `@` is rejected — `@media`/nested rules/`@import`
are impossible through this field on purpose (it's spliced verbatim into a real `<style>` tag served
to every end user).

## Practical guidance for anything generating App Studio style values

1. Default to `style`, never `css`, unless the property genuinely has no `StyleProperties` field.
2. Box-model/border-radius values are **numbers** (pixels implied), never unit strings.
3. `width`/`height`/etc. ARE strings — box-model spacing is not.
4. Never emit `{`, `}`, or `@` inside `css` — silently rejected, not just discouraged.
5. Structured `style` composes correctly Section → Widget (later/more-specific wins); `css` text does
   not cascade the same way — it's a flat injected rule per instance.

## Known gap

No `top`/`left`/`right`/`bottom` free-position offset fields exist today — a canvas/free-position
mode was designed (not yet built as of the source scan date) that would add these additively.

## The structured style system CANNOT do hover states, animations, or nested-selector CSS — this is a hard limit, not a missing property

Confirmed 2026-09-11, root-caused by reading `cssInjector.ts` directly: `injectScopedCss` wraps
`css` as `.as-style-{hash} { ${css} }` — literally ONE rule, ONE element, no selectors of any kind
possible. `style: StyleProperties` is richer (free-text values for `transform`/`transition`/
`boxShadow`/`backgroundImage` gradients/`filter`) but still only ever styles the single element it's
attached to — no `:hover`, no `::before`, no `@keyframes`, no `> .child` targeting. **Neither Level
2 (`sectionContainer`) nor Level 3 (`widgetContainer`) style slot can produce a hover effect, a
scroll-reveal animation, or a card-hover-lift — full stop, regardless of how the values are
expressed.** Don't spend time trying to fake these through the structured system; they are
structurally impossible there.

**The real answer for anything needing hover/animation/nested-selector styling is
`content` widget's `allowScripts: true` config flag** — this switches `ContentSanitizer`'s DOMPurify
call from a small tag/attr allowlist to its full default profile, which DOES allow real `<style>`
tags with real selectors, pseudo-classes, and `@keyframes`. This is a genuine, already-shipped
platform feature (`ContentWidgetConfig.allowScripts?: boolean`, read `ContentSanitizer.ts` to
confirm), not a workaround — but nothing in the Designer UI frames it as a styling feature (it reads
as a security/scripting toggle), so it's effectively undiscovered by normal use. When building a
polished custom section (a card grid with hover, a portfolio thumbnail reveal-on-hover, a sticky
nav), use ONE content widget per section with `allowScripts: true` and a single `<style>` block
**scoped under one unique wrapper class per widget** — every content widget's `<style>` tag lands in
the same page `<head>`, so an unscoped/bare selector (`h2 {}`, `a {}`) leaks into every other
section on the page. Reserve the structured `sectionContainer`/`widgetContainer` style slots for
simple whole-element background/spacing banding between sections, and `AppSection.widgetLayout` (a
sibling field to `sectionContainer`, NOT part of it — see the 2026-09-04 `SECTION_CHILD_FILL_STYLE`
fix in `AppPlayer.tsx`) for arranging multiple widgets in one section as a flex row.

## App Studio Theming — the `--app-var-*` token contract (added 2026-09-12, app-studio-theme feature)

When a `content` widget's `<style>` block (the `allowScripts: true` escape hatch above) needs a
color, font, radius, shadow, or spacing value that should belong to the APP's overall look rather
than being hardcoded to one specific hex/px value forever, reference the app-wide theme token
instead of a literal — this is the single most important habit for any future widget generation to
pick up from this section:

```css
/* Don't: */ .my-card { background: #f59e0b; color: #0f172a; }
/* Do:    */ .my-card { background: var(--app-var-color-primary); color: var(--app-var-bg); }
```

The full, fixed 12-token set (source of truth: `app-handlers-core/src/types/ThemeTokens.ts`):

| Token | Typical use |
|---|---|
| `--app-var-color-primary` | brand accent — buttons, links, highlighted text, active states |
| `--app-var-color-secondary` | a second accent, used more sparingly than primary |
| `--app-var-bg` | the app's base/darkest background |
| `--app-var-bg-panel` | a one-step-lighter panel/section background (alternating sections, cards) |
| `--app-var-text` | primary/heading text color |
| `--app-var-text-muted` | secondary/caption/label text color |
| `--app-var-border` | hairline borders, dividers |
| `--app-var-font-family` | body font stack |
| `--app-var-font-heading` | heading font stack (can differ from body) |
| `--app-var-radius` | corner radius for buttons/cards/inputs |
| `--app-var-shadow` | box-shadow for elevated elements |
| `--app-var-space-unit` | base spacing unit for gap/padding rhythm |

**For an alpha-blended/translucent variant of a token (a glow, a subtle border, an overlay)**, do
NOT hand-write `rgba(<token's literal RGB>, 0.2)` — the token holds a full CSS value (hex, a
gradient stop, a keyword), not bare R,G,B components, so `rgba(var(--app-var-color-primary), 0.2)`
does not work. Use `color-mix()` instead:

```css
border: 1px solid color-mix(in srgb, var(--app-var-color-primary) 30%, transparent);
```

**Do NOT invent a 13th token or reuse a shared-tool-chrome variable** (`--bg-dark`, `--text-primary`,
`--color-bg-primary`, etc. — see `design.md` §1 for why three of those already collide with each
other outside App Studio). The 12-token list above is deliberately fixed and namespaced; a widget
needing something the list doesn't cover should fall back to a literal value for that one property,
not smuggle in a new custom-property name.

**These variables are only meaningful inside an App Player-rendered app** (they resolve via
`AppPlayer.tsx`'s `[data-theme-scope="app"]`/`[data-page-id]` cascade — see
`app-handlers-generic/src/hooks/themeInjector.ts`) — never reference them from Designer chrome,
Flow Studio, or any other tool surface outside App Studio's own runtime.

Full design/build history: `Documentation\WorkManagement\app-studio-theme\{design.md,plan.md}`.
