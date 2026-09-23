# Style Properties — the Structured Style Builder System

Source of truth: `app-handlers-core/src/types/StyleSlot.ts` +
`atlas-forms/packages/designer-components-react/src/components/StyleBuilderPanel/types.ts`, scanned
2026-08-30. **Always prefer emitting the structured `style` object over the `css` escape hatch** — it
is typed, enumerable, and cannot be rejected by the sanitizer (see below); `css` should be reached for
only when `StyleProperties` genuinely has no matching field.

## StyleSlotValue — the one shared value shape at every level

```ts
interface StyleSlotValue { css?: string; style?: StyleProperties; }
```

| Level | Lives on | Field names |
|---|---|---|
| 1 — whole App | `App.styleConfiguration` | `siteContainer`, `siteBackground` (edit-side only today, not yet rendered) |
| 2 — a Section | `AppSection` (rides in `layout` JSON, no own DB column) | `sectionContainer`, `sectionBackground` |
| 3 — a Widget placement | `AppWidgetRecord.styleConfiguration` | `widgetContainer`, `widgetBackground` (wired), + 8 more named-but-unwired slots: `widgetHeader`/`widgetFooter`/`widgetBody`/`widgetContent`/`widgetImage`/`widgetText`/`widgetIcon`/`widgetTitle` |

## `style: StyleProperties` — full field list (~90 properties, all optional, camelCase)

Cast straight to `React.CSSProperties` at render time — no transformation, no shorthand (e.g.
`paddingTop`/`paddingRight`/`paddingBottom`/`paddingLeft` as four numbers, never a `padding` shorthand
string).

**Color/background**: `backgroundColor`, `color`, `borderColor`, `textShadow`, `backgroundImage`
(URL string), `backgroundSize`, `backgroundPosition`, `backgroundRepeat`
(`'repeat'|'no-repeat'|'repeat-x'|'repeat-y'`), `backdropFilter`, `filter`.

**Typography**: `fontFamily`, `fontSize` (number), `fontWeight`
(`'normal'|'medium'|'semibold'|'bold'|number`), `fontStyle` (`'normal'|'italic'`), `lineHeight`
(number), `letterSpacing` (number), `textAlign` (`'left'|'center'|'right'|'justify'`),
`textTransform`, `textDecoration`, `textOverflow` (`'clip'|'ellipsis'`), `whiteSpace`, `wordBreak`.

**Box model** (all numbers, no shorthand): `paddingTop`/`Right`/`Bottom`/`Left`,
`marginTop`/`Right`/`Bottom`/`Left`, `gap`, `rowGap`, `columnGap`.

**Sizing**: `width`, `minWidth`, `maxWidth`, `height`, `minHeight`, `maxHeight` (all strings, e.g.
`"480px"`/`"100%"`), `aspectRatio` (string), `objectFit`, `objectPosition`.

**Overflow**: `overflow`, `overflowX`, `overflowY` (`'visible'|'hidden'|'scroll'|'auto'`).

**Border**: `borderWidth` + 4 per-side `borderTopWidth`/etc. (numbers), `borderStyle` + 4 per-side
(`'none'|'solid'|'dashed'|'dotted'|'double'`, per-side variants omit `'double'`), `borderTopColor`/
etc. (4, no shorthand `borderColor` per-side — that's the single `borderColor` field above, applies to
all sides at once), `borderRadius` + 4 corner-specific (all numbers, never a CSS string like
`"16px"`).

**Outline/shadow/effects**: `outlineWidth`, `outlineStyle`, `outlineColor`, `outlineOffset`,
`boxShadow` (string, e.g. `"0 8px 24px rgba(15,23,42,0.08)"`), `opacity` (number 0-1), `transform`
(string), `transition` (string), `cursor` (string), `userSelect`, `pointerEvents`, `clipPath`.

**Layout**: `display` (`'block'|'flex'|'grid'|'inline-block'|'inline-flex'`), `flexDirection`,
`justifyContent`, `alignItems`, `flexWrap`, `position` (`'static'|'relative'|'absolute'|'sticky'|
'fixed'`), `zIndex` (number).

**NOT present today** (real gap, not an oversight): `top`/`left`/`right`/`bottom` offset fields — a
free-position widget/canvas-mode design was written this session (see `architecture.md` §8, Task 3 in
`design-and-plan.md`) that adds exactly these four, additively, when built.

## `css: string` — the escape hatch, deliberately narrow

Raw CSS **declarations** (property:value pairs), never selectors or rules — e.g.
`"color: red; padding: 8px;"`. Scoped to a deterministic hashed className at inject time so identical
text across instances shares one rule.

**Hard sanitizer rule, security-load-bearing, not a bug to work around**: the sanitizer
(`cssInjector.ts`'s `sanitizeScopedCss`) actively rejects any text containing `{`, `}`, or `@`. This
means `@media`/`@container`/nested rules/`@import` are all impossible through this field, on purpose
— the field is free text an app-builder-level user enters, spliced verbatim into a real `<style>` tag
served to every end user of the live app. **Do not attempt to route responsive/breakpoint styling
through `css` even as a workaround** — the correct mechanism is a genuinely separate, structured type
(designed this session, `ResponsiveStyleValue{base,tablet?,mobile?}` using CSS Container Queries —
see `architecture.md` §8; not yet built).

## Practical guidance for anything generating App Studio style values

1. Default to `style`, never `css`, unless the specific property genuinely has no `StyleProperties`
   field.
2. Box-model and border-radius values are **numbers** (pixels implied), never strings with units.
3. `width`/`height`/`minWidth`/etc. ARE strings (can be `"100%"`, `"480px"`, etc.) — box-model spacing
   is not.
4. Never emit `{`, `}`, or `@` inside a `css` field — it will be silently rejected at apply time, not
   just discouraged.
5. Structured `style` values compose correctly through Section → Widget levels (later, more-specific
   levels' inline styles win via normal CSS specificity — inline always beats an injected class); `css`
   text does not cascade the same way, it's a flat injected rule per instance.
