# Testing the Workflow Category Widget (`widgetType: 'workflow-template-category'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Workflow Category · **Description**: "A grid of every agent in one execution-template
category (name/description/icon, Execute or Chat Now per agent)."

Like the Workflow Agent widget, but bound to a whole **category** (`executionTemplateCategoryID`)
instead of one template — renders every enabled agent in that category as a grid of cards, each
with its own independent Execute/Chat Now button.

## 2. How to add one

Same preferred pattern as Workflow Agent: open the Toolbox drawer's Widgets tab, expand
**"Workflow Agent Categories"**, drag the category you want directly onto a section. The Add Widget
modal's own version requires a bare numeric `ExecutionTemplateCategoryID` — prefer the Toolbox drag
path.

## 3. Functional test checklist

- [ ] Drag a real category onto a section — confirm it renders a grid with EVERY enabled agent in
  that category (cross-check the count against what you see for the same category in Flow
  Studio/WorkDesk).
- [ ] Click Execute/Chat Now on two DIFFERENT cards within the same widget — confirm they act
  independently (one executing doesn't disable/affect the other).
- [ ] Edit to rebind to a different category — confirm the grid updates to the new category's
  agents.
- [ ] Delete and confirm removal persists.
- [ ] Check in Preview / App Player.

## 4. Usability checklist

- [ ] With a category that has MANY agents, does the grid stay usable (proper wrapping/scrolling),
  or does it become an unwieldy wall of cards?
- [ ] With a category that has ZERO enabled agents (if you can find/create one), does the widget
  show a sensible empty state, or a confusing blank area?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — grid should reflow to fewer columns at narrower widths, not overflow
  or force horizontal scrolling.
- [ ] Dark theme — consistent card styling with the single Workflow Agent widget's own card design
  (same visual language expected).

## 6. Widget-specific edge cases

- [ ] Same Execute/Chat Now-per-card Type-field caveat as the Workflow Agent widget applies here too
  — see that widget's README §6 for the full explanation, it applies per-card in this grid.
- [ ] If a category contains a MIX of Execute-type and Chat-type agents, confirm each card correctly
  shows its own right button — not all cards defaulting to the same one.
