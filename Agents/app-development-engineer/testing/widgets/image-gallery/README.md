# Testing the Image Gallery Widget (`widgetType: 'image-gallery'`)

> Read `../common/resource.md` first — especially §4 (Public/Private assets), this widget's whole
> job is querying and displaying PUBLIC assets live, so that boundary matters a lot here.

## 1. What it is

**Label**: Image Gallery · **Description**: "A live, filterable collection of public images —
Classic Grid, Masonry, Fade/Slide Carousel, or Ken Burns."

Unlike the single Image widget (one fixed URL), this widget performs a LIVE QUERY at render time
against digital-assets-library's public assets, filtered by optional criteria, and displays however
many match in one of several visual themes.

## 2. How to add one — the creation wizard

Add Widget → New Widget → **Image Gallery**. This is the one widget type with a **guided
multi-step wizard** instead of a flat form straight away:

1. **Step 1 — Theme**: clickable preview cards for **Classic Grid**, **Masonry**, **Fade Carousel**,
   **Slide Carousel**, **Ken Burns**. Pick one.
2. **Step 2 — Size/Automation**: for grid-style themes, a **Columns** setting; for carousel-style
   themes, an **Automation** speed preset (named presets like Off/Relaxed/Standard/Energetic rather
   than raw seconds).
3. **Land on the full Configuration screen** — your Step 1/2 choices are pre-filled here but still
   editable, plus the real filter/content fields: optional **Document Type** and **Document
   Category** (real searchable lookup fields, not raw ID boxes — confirm this), optional **Media
   Category** override, optional **Title**, **Max Items** cap.
4. A **Back** button should let you return to the previous wizard step without losing your Step 1
   choice.

**Note**: editing an EXISTING Image Gallery widget (Widget Details, not creation) should skip the
wizard entirely and go straight to the flat Configuration screen — confirm this is actually the
case; the wizard is meant to be a first-time-setup aid only.

## 3. Functional test checklist

- [ ] Complete the wizard with each of the 5 themes at least once across your testing, confirm each
  produces a visibly different, working rendering (not just a text label under identical layout).
- [ ] With NO filters set at all (empty Document Type/Category/Media Category), confirm it shows
  ALL public images currently in the library — this "zero config = works" behavior is a deliberate
  design goal, verify it actually holds.
- [ ] Set a real Document Category filter, confirm the gallery correctly narrows to only matching
  assets.
- [ ] Set Max Items to a small number (e.g. 2) with more than 2 matching assets available — confirm
  it correctly caps the display.
- [ ] Edit an existing gallery — confirm it goes straight to the flat config (no wizard replay), and
  that changing the theme there still visibly updates the rendering.
- [ ] Delete, confirm removal persists.
- [ ] Check in Preview / App Player — confirm the SAME live query/results appear there (this proves
  the "live query" part is really live server-side, not something faked in the Designer only).

## 4. Usability checklist

- [ ] Are the 5 theme preview cards in Step 1 visually distinct enough at a glance to make an
  informed choice, or do they look too similar to tell apart before picking?
- [ ] With zero matching assets (e.g. a filter combination that matches nothing), does the gallery
  show a clear, non-broken-looking empty state?

## 5. Styling/visual checklist — test EVERY theme, this is the core of what makes this widget

- [ ] **Classic Grid**: uniform grid, correct column count per your Columns setting, reflows fewer
  columns on Tablet/Mobile.
- [ ] **Masonry**: genuinely variable-height columns (Pinterest-style), not just a grid with
  padding — confirm images of different aspect ratios actually produce a staggered layout.
- [ ] **Fade Carousel**: one image at a time, smooth cross-fade transition between images (not an
  abrupt jump-cut).
- [ ] **Slide Carousel**: one image at a time, horizontal slide transition, with visible prev/next
  arrows AND dot indicators — test clicking both.
- [ ] **Ken Burns**: confirm a real, continuous slow pan/zoom effect is actually happening on the
  current image (not a static image mislabeled as this theme).
- [ ] Automation presets (carousel themes) — confirm the SPEED actually differs noticeably between
  e.g. the slowest and fastest presets, not identical timing under different labels.
- [ ] Dark theme consistency across all 5 themes.

## 6. Widget-specific edge cases — security is the priority here

- [ ] **Critical security check**: upload a test image marked **Private**, then load/reload a
  no-filter Image Gallery widget — confirm the Private image NEVER appears, only Public ones. This
  is the single most important thing to verify for every gallery widget — treat any failure here as
  a high-priority, immediate-report finding (see `../common/resource.md` §4/§5).
- [ ] Confirm Document Type/Document Category filter fields are real searchable lookups (type to
  search, pick from a real list of existing categories/types) — NOT plain numeric ID boxes. If you
  see a bare number field instead, that's a regression worth reporting.
- [ ] Test with a MediaCategory override that doesn't match "Image" (if the field allows it) —
  confirm the widget's behavior in that mismatched-override case is at least sensible (empty result,
  not a crash) even if it's a slightly unusual configuration.

## 7. Known limitation — full-resolution images, no thumbnails

Same as the single Image widget: every gallery item downloads the **original uploaded file**, even
in Grid/Masonry themes rendering it as a small tile — no server-side thumbnail generation exists
yet. `loading="lazy"` is applied to gallery images so offscreen ones defer loading, but this doesn't
reduce transferred bytes for images that ARE visible. Not a bug — a known, accepted gap. A gallery
with many large originals will genuinely be slower to fully load than a real thumbnail-backed
gallery would be; that's expected today.
