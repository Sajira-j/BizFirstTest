# Responsive Breakpoint Editing — Design

Status: Design only, no code written. Companion to the other `wix-style-design` docs (this one
covers Task 6: per-device style editing, Wix-style).

## 1. Current state (verified against source)

**Data model** (`app-handlers-core/src/types/StyleSlot.ts`):

```ts
export interface StyleSlotValue {
  css?: string;
  style?: StyleProperties;
}
```

`StyleProperties` (`atlas-forms/packages/designer-components-react/src/components/StyleBuilderPanel/types.ts`)
is a single flat object — ~90 named CSS-like properties (`paddingTop`, `borderRadius`, `display`,
`flexDirection`, `width`, etc.), each a plain scalar (`string | number | enum`). There is no
per-device variant anywhere in this type, and no breakpoint concept exists in `AppSection`,
`AppWidget`, or `App` — every level (`SiteStyleConfig`, `WidgetStyleConfig`, and the ad-hoc
`sectionContainer`/`sectionBackground` slots on `AppSection`) stores exactly one `StyleSlotValue`
per named region.

**Render path** (`app-handlers-generic/src/hooks/useStyleSlot.ts`):

```ts
const style = (value?.style ?? {}) as React.CSSProperties;
```

`style` is cast straight to inline `React.CSSProperties` — no transformation, no media query
capability. Inline styles fundamentally cannot express `@media` rules; that is a hard CSS
constraint, not an implementation gap.

**The `css` escape hatch is also blocked from expressing breakpoints, on purpose.**
`cssInjector.ts`'s `sanitizeScopedCss` rejects any text containing `{`, `}`, or `@`
(`UNSAFE_CSS_PATTERN = /[{}<]|@|expression\s*\(|.../`). This is a deliberate security boundary —
the injected `css` is free text an author (not necessarily a trusted developer) can enter, and it's
spliced verbatim into a real `<style>` tag served to every end user of the live app-player runtime.
Allowing `@media` would also allow `@import url(evil)` and rule-escaping via a bare `}`. **This
means breakpoints cannot be bolted onto the existing `css` field even as a stopgap** — a new,
separate mechanism is required whose input is always structured (numbers/enums from the Style
Builder UI), never free text, so it carries no injection risk by construction.

**Editing UI** (`app-studio-designer-components-react/src/style/`):
`StylePropertiesTab` renders one `StyleSlotEditor` per named slot (Site/Section/Widget style tabs
all reuse this one component). Each `StyleSlotEditor` shows a `StyleSlotRow` trigger that opens the
shared Flow Studio `StyleBuilderPanel` popover (`onApply(styles: StyleProperties)` — one flat
object in, one flat object out) plus a plain "Custom CSS" textarea for the `.css` half. Nothing in
this chain is breakpoint-aware.

**Confirmed lack of responsive handling in real content.** Qoboto's own `Configuration.layout` (app
1526) hard-codes fixed pixel widths with no fallback: `hero-left`/`hero-right` are each
`"width":"480px"`, the `recent-websites-grid`/`product-grid-cards` cards are `"width":"280px"`,
`web3-features-grid`/`template-gallery-grid` cards are `"width":"320px"`, and
`trusted-partners-grid` cards are `"width":"220px"`. On a 375px mobile viewport, `hero-left` alone
(480px, plus 48px section padding each side per `hero`'s `sectionContainer`) already overflows the
device width by more than 200px. This is the concrete, present-day cost of having no breakpoint
system — not a hypothetical.

## 2. Data model change

Extend `StyleSlotValue.style` to accept either the current flat shape (unchanged) or a
per-breakpoint map, discriminated structurally (no version flag needed — see backward-compatibility
below):

```ts
export interface ResponsiveStyleValue {
  base: StyleProperties;              // required — desktop-and-up, and the fallback for any
                                       // device that has no narrower override
  tablet?: Partial<StyleProperties>;  // ≤ TABLET_MAX_WIDTH, overrides `base`
  mobile?: Partial<StyleProperties>;  // ≤ MOBILE_MAX_WIDTH, overrides `base` (and `tablet`,
                                       // if `tablet` is also set — see cascade rule below)
}

export interface StyleSlotValue {
  css?: string;
  style?: StyleProperties | ResponsiveStyleValue;
}
```

Discriminate at read time with a single structural check: `'base' in value.style` → responsive map;
otherwise → today's flat `StyleProperties` (no real-world `StyleProperties` object has a `base`
key — it isn't one of the ~90 declared properties — so this is unambiguous and needs no schema
version bump).

**Cascade rule**: `mobile` overrides `tablet` overrides `base`, per-property (not per-object) —
mirrors how CSS's own mobile-first `@media` cascade behaves, and matches what a designer expects
("I only changed the font size for mobile; everything else should still follow whatever tablet
says, or base if tablet didn't touch it either").

## 3. Breakpoint cutoffs

Two breakpoints (tablet, mobile), three zones — matches Wix Editor's own three-device model (not
five/six-tier frameworks like Tailwind's, which is more granularity than a visual-only Style
Builder needs):

- **Desktop (`base`)**: ≥ 1024px
- **Tablet**: 768px–1023px (`@media (max-width: 1023px)`)
- **Mobile**: ≤ 767px (`@media (max-width: 767px)`)

768/1024 are the two most common device-class boundaries in real analytics data (iPad portrait is
768px wide; 1024px is the standard "small laptop / large tablet landscape" cutoff) and are already
what Wix, Squarespace, and Webflow all converge on. Picking anything more exotic would fight
designer intuition for no benefit — this is a case where "match the incumbent" is the right call,
not a novel choice to defend.

## 4. Editing UI

**Device switcher**: a 3-button segmented control (Desktop / Tablet / Mobile icons — reuse the
`Accordion`/toolbar icon conventions already shipped this session) added to `StyleSlotEditor`'s
header, next to the slot label. Selecting a device:
- Filters which `StyleProperties` fields the `StyleBuilderPanel` popover edits (still edits one
  flat object at a time — the popover itself does NOT need to become breakpoint-aware; it's handed
  `value.style.base` / `value.style.tablet ?? {}` / `value.style.mobile ?? {}` depending on
  selection, and `onApply` writes back to that same key).
- Shows a small "inherits from Desktop" hint chip when Tablet/Mobile has no explicit override for a
  field that Desktop sets (read-only visual cue, not an editable state) — this is what stops users
  from thinking mobile has "nothing" when it's silently inheriting.

**Where the switcher lives**: at the `StyleSlotEditor` level (per-slot), not globally at the top of
the whole Designer — a section's container and a widget's title might need different breakpoints to
diverge, and a global switcher would force every slot into the same edit-device even when only one
slot actually needs a mobile override. This does mean a user editing multiple slots for the same
device has to reselect "Mobile" per slot; acceptable given the alternative (a single wrong device
selection silently affecting slots the user didn't intend to touch) is worse.

**Canvas preview width toggle**: add a matching 3-button Desktop/Tablet/Mobile control to
`LivePreviewPanel`'s own toolbar (not `StyleSlotEditor` — this one IS global, since the whole canvas
renders at one width at a time). Selecting a device sets the preview `<div>` wrapper's `width` to a
fixed value (1280px / 834px / 390px — representative, not exact-device, matches how Wix's own
device toggle works) and lets normal CSS `@media` queries (now real, see section 5) do the rest;
the Designer does not need to fake `window.innerWidth` for `AppPlayer` — a narrower **container**
combined with real `@media (max-width: ...)` rules on the injected classes produces the correct
preview as long as the breakpoints are max-width and the fixed-width container is narrower than the
threshold (verified: our thresholds are viewport-width media queries, which respond to the
**window's** width, not the container's — see the caveat in section 6 below; this is resolved by
using `ResizeObserver`-driven container queries instead of `@media`, not `@media` after all — see
revised section 5).

## 5. Rendering engine change (`cssInjector.ts` / `useStyleSlot.ts`)

**Plain `@media (max-width: ...)` cannot be previewed correctly inside a fixed-width Designer
canvas panel** — `@media` queries respond to the actual browser **viewport** width, not the width of
the `<div>` the preview is embedded in. The Designer's `LivePreviewPanel` renders inside a canvas
panel that is narrower than the full browser window even in "mobile preview" mode (there's a
toolbar, a details sidebar, etc. around it) — so a real `@media (max-width: 767px)` rule would never
fire inside the Designer even when the preview column is visually narrow, while `AppPlayer`'s real
end-user runtime (which genuinely resizes with the browser) would work correctly. This asymmetry
would make the Designer's "Mobile" preview lie.

**Design choice: CSS Container Queries, not `@media`.** Modern container queries
(`@container (max-width: ...)`, supported in every browser this codebase already targets per its
Vite/React 18 baseline — no polyfill needed for Chromium-based dev/test, which `claude-in-chrome`
also confirms this session runs on) query the nearest ancestor with `container-type: inline-size`,
not the viewport. This makes the SAME injected rule correct in both places:
- In `AppPlayer`'s real runtime, wrap the top-level rendered app root in one `container-type:
  inline-size` div (a single, one-time change to `AppPlayer.tsx`'s root wrapper) — it then
  naturally tracks the real browser width end-to-end.
- In the Designer's `LivePreviewPanel`, the existing preview `<div>` wrapper (already a container
  the app renders inside) gets `container-type: inline-size` too, and the width-toggle from section
  4 sets that container's fixed pixel width directly — the exact same `@container` rules now fire
  correctly at design time, matching runtime pixel-for-pixel.

**Concrete `cssInjector.ts` change**: `injectScopedCss` currently emits one rule:
```css
.as-style-{hash} { ${css} }
```
For a `ResponsiveStyleValue`, add a new function (not a change to the existing free-text path,
which stays exactly as-is and keeps rejecting `{`/`}`/`@` — the responsive path's input is never
free text, so it doesn't go through `sanitizeScopedCss` at all):

```ts
export function injectResponsiveStyle(value: ResponsiveStyleValue): string {
  const className = `as-rstyle-${hashString(JSON.stringify(value))}`;
  if (injectedRules.has(className)) return className;

  const toDecl = (props: Partial<StyleProperties>) =>
    Object.entries(props).map(([k, v]) => `${cssPropName(k)}: ${cssValue(k, v)};`).join(' ');

  const rules = [`.${className} { ${toDecl(value.base)} }`];
  if (value.tablet) rules.push(
    `@container (max-width: 1023px) { .${className} { ${toDecl(value.tablet)} } }`);
  if (value.mobile) rules.push(
    `@container (max-width: 767px) { .${className} { ${toDecl(value.mobile)} } }`);

  const styleEl = document.createElement('style');
  styleEl.setAttribute('data-as-style', className);
  styleEl.textContent = rules.join('\n');
  document.head.appendChild(styleEl);
  injectedRules.set(className, styleEl);
  return className;
}
```

`cssPropName` (camelCase → kebab-case, e.g. `paddingTop` → `padding-top`) and `cssValue` (append
`px` to the same unitless-exempt set React's own style engine exempts — `lineHeight`, `opacity`,
`zIndex`, `fontWeight`, `flexGrow`/`flexShrink`, `order`, `zIndex`) are the only new small helpers
needed; both are pure functions over a closed, already-known property list (`StyleProperties`
itself), not user text, so there is no sanitization concern — the values are whatever the
`StyleBuilderPanel` popover's own typed controls (color pickers, number inputs, enum dropdowns)
produced, which can't contain `{`/`}`/`@` in the first place.

**`useStyleSlot.ts` change**: today it returns `{ style: value.style as CSSProperties, className }`
(base object goes inline, `css` text goes to a class). For the responsive shape, base-level values
still go inline (cheapest, no extra rule needed for the desktop case, and inline always wins
over the `@container`-scoped base rule anyway per CSS specificity — so put `base` inline AND skip
emitting it in the injected rule's first block, using the class only for the tablet/mobile
overrides):

```ts
export function useStyleSlot(value: StyleSlotValue | undefined): UseStyleSlotResult {
  const styleValue = value?.style;
  const isResponsive = !!styleValue && 'base' in styleValue;

  const cssClassName = useMemo(() => {
    if (value?.css) return injectScopedCss(value.css);          // unchanged path
    if (isResponsive) return injectResponsiveStyle(styleValue as ResponsiveStyleValue, /* baseInline: true */);
    return undefined;
  }, [value?.css, styleValue, isResponsive]);

  const inlineStyle = (isResponsive ? (styleValue as ResponsiveStyleValue).base : styleValue ?? {}) as React.CSSProperties;
  return { style: inlineStyle, className: cssClassName };
}
```

The one caller-visible change: the DOM node needs `className` applied even when there's no `.css`
text, whenever `style` is responsive — `StyledSlot.tsx` (the sole consumer) already applies both
`style` and `className` unconditionally today, so **no change is needed there** — confirmed by
reading its current unconditional `<div style={style} className={className}>` usage.

## 6. Backward compatibility

- **Zero data migration.** Every existing `StyleSlotValue.style` in the DB (qoboto's included) is a
  flat `StyleProperties` object with no `base` key. The `'base' in value.style` discriminator
  correctly routes 100% of existing data down the unchanged flat path — `useStyleSlot` behaves
  byte-for-byte as it does today for every app that has never opened the new device switcher.
- **Opt-in only.** A section/widget only becomes "responsive" the moment an author explicitly picks
  Tablet or Mobile in the new device switcher and changes something — at that point (and only then)
  `StyleSlotEditor`'s `onChange` starts writing the `{base, tablet?, mobile?}` shape instead of the
  flat one. Until that first edit, nothing about existing content changes.
- **`StyleBuilderPanel` itself needs zero changes** — it already edits one flat `StyleProperties`
  object per call; `StyleSlotEditor` decides WHICH flat object (`base`/`tablet`/`mobile`) to hand it
  based on the new device switcher's selection. This keeps the change's blast radius inside
  `app-studio-designer-components-react`'s own `style/` folder plus the two `app-handlers-generic`
  files above — `@atlas-forms/designer-components-react` is untouched.
- **`AppPlayer`'s root `container-type: inline-size` addition** is the one runtime change outside
  the style files. It's inert for every app that has zero responsive style values (container-type
  only affects `@container`-querying descendants, of which there would be none) — safe to ship
  ahead of/independent from the rest.

## 7. Interaction with Task 3 (free-form drag-and-drop positioning)

Flagged, not designed here: if Task 3 introduces per-widget absolute `{x, y}` coordinates, those
coordinates would need their OWN per-breakpoint variants too (a widget positioned at x:400 on a
1280px canvas needs a different x on a 390px canvas, unlike flow properties which reflow
automatically) — i.e. `ResponsiveStyleValue`-shaped position data, not just style properties. Design
Task 3's positioning model with this in mind (make position itself a breakpoint-varying field from
day one) rather than retrofitting it later — but the actual design of that interaction belongs in
Task 3's own doc, not here. Sequencing recommendation: ship this (Task 6) BEFORE Task 3, since
Task 3's own effort estimate should already account for needing responsive position data, not
discover it as a surprise mid-implementation.

## 8. Phased implementation plan

**Phase 1 — Data model + render engine (no UI yet).**
- Add `ResponsiveStyleValue` type, `injectResponsiveStyle`, `cssPropName`/`cssValue` helpers,
  `useStyleSlot` branch, `AppPlayer` root `container-type: inline-size`.
- Unit tests: `cssInjector.test.ts` gains cases for the container-query output shape, the
  base-inline/tablet-mobile-in-class split, and idempotency (same hash → no duplicate `<style>`).
- Effort: 0.5–1 day (small, well-isolated files; the existing `cssInjector.test.ts` pattern to
  extend already exists).

**Phase 2 — Editing UI.**
- Device switcher in `StyleSlotEditor`, wiring to read/write the right sub-object.
- "Inherits from Desktop" hint chip.
- Effort: 1–1.5 days (one component's worth of new UI, reusing existing `StyleBuilderPanel`
  unchanged).

**Phase 3 — Canvas preview width toggle.**
- `LivePreviewPanel` toolbar control + fixed-width container swap.
- Effort: 0.5 day (small, self-contained; no store changes needed — purely a local UI state driving
  a wrapper `<div>`'s width).

**Phase 4 — Verification pass.**
- Live-test: qoboto's own `hero-left`/`hero-right` 480px cards, set a Mobile override
  (`width: '100%'`), confirm the Designer's Mobile preview AND the real `AppPlayer` (resized
  browser window) both reflow correctly and identically.
- Effort: 0.5 day.

**Total: ~2.5–3.5 days** for one engineer familiar with this codebase's existing Style Builder
chain (all four phases touch code already read and understood during this design pass — no
unknowns remain that would require exploratory spikes).

## 9. Explicit stoppers / risks

- **None blocking.** Container queries are supported in every browser this workspace's dev/test
  tooling targets (verified this session's browser automation runs on Chromium). If IE11 or a
  legacy WebView target is ever added to this product's support matrix, container queries would
  need a fallback (`@media` on `AppPlayer`'s real runtime only, accepting the Designer-preview
  asymmetry described in section 5) — not a concern today, called out so it isn't rediscovered
  later as a surprise.
- **Minor**: `injectResponsiveStyle`'s hash key is `JSON.stringify(value)` — key ordering inside
  each `Partial<StyleProperties>` object must be stable for the hash to correctly dedupe identical
  values across instances. `StyleBuilderPanel`'s `onApply` always produces objects via the same
  code path (not user-typed key order), so this is a non-issue in practice, but worth a one-line
  code comment when implemented so a future refactor doesn't accidentally introduce
  non-deterministic key order upstream.

## Build Progress

**2026-08-30 — Phases 1-3 shipped, Phase 4 (live verification) deferred.**

- **Phase 1 (data model + render engine)**: `ResponsiveStyleValue` type (`app-handlers-core`),
  `injectResponsiveStyle`/`cssPropName`/`cssValue` (`app-handlers-generic/cssInjector.ts`),
  `useStyleSlot` responsive branch, `AppPlayer` root `container-type: inline-size` (both wrapper
  branches). `mergeStyleSlotValues`'s flat shallow-spread was a real bug this design didn't
  anticipate — fixed with a dedicated `mergeResponsiveInto` (merges each device independently) so
  a responsive `sectionContainer` combined with a flat `sectionBackground` — a real, live path via
  `AppPlayer.tsx`'s existing `mergeStyleSlotValues(section.sectionContainer,
  section.sectionBackground)` call — doesn't corrupt either shape. 4 new unit tests in
  `cssInjector.test.ts` (container-query shape, unitless props, no-override devices omit their
  block, idempotency) — 15/15 passing.
- **Phase 2 (editing UI)**: device switcher + inherit hint in `StyleSlotEditor.tsx`, exactly as
  designed — `StyleBuilderPanel` itself untouched. Real TS lesson worth flagging for future
  similar work: a boolean narrowing flag doesn't keep a re-read of the original expression narrowed
  in TypeScript — had to narrow into one local once and derive everything from that local, not from
  re-checking the original union-typed expression repeatedly.
- **Phase 3 (canvas preview width toggle)**: Desktop/Tablet/Mobile/Full-width toggle in
  `LivePreviewPanel`'s own toolbar, exactly as designed (representative widths, local state only).
- **Phase 4 (verification pass)**: **deferred, not skipped** — per Binoy's explicit direction this
  round ("build 6, 4b, 3, 7, 8... we can do a complete verification at the end"), live-browser
  testing against qoboto's real `hero-left`/`hero-right` 480px cards was intentionally not chased
  this pass — the dev box was memory-constrained/backend-flaky for most of this session, and Binoy
  wants one consolidated verification pass at the end rather than per-task fights with an unstable
  backend. `tsc --noEmit` clean across all 3 touched packages
  (`app-handlers-core`/`app-handlers-generic`/`app-studio-designer-components-react`) is this
  pass's actual correctness bar, per that same direction.
- **No blockers found.** Task 3 (free-form drag-and-drop, queued immediately after this) can
  proceed — the `{base, tablet?, mobile?}` schema shape this design settled on is the one Task 3's
  own doc asked to sync on before either shipped.
- No git commits made.
