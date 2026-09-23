# Testing the Page Navigation Widget (`widgetType: 'page-navigation'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Page Navigation · **Description**: "A placeable menu of app pages — expandable/
collapsible for nested pages, vertical or horizontal."

Renders a real navigation menu of the app's own pages, driven by the app's actual page tree (not
manually configured links). Note: once an app places one of these, the legacy fixed top/side
`AppNavMenu` bar stops rendering — this widget replaces it, it doesn't add to it.

## 2. How to add one

Safe-default type — drag directly from the Toolbox Widgets tab (instant-places with default
"vertical" orientation), or Add Widget → New Widget → Page Navigation, choose Vertical or
Horizontal.

## 3. Functional test checklist

- [ ] Drag-place, confirm it renders a real menu listing the app's actual pages (create 2-3 test
  pages first via the Pages drawer if the app doesn't have enough to see a meaningful menu).
- [ ] Create a nested page (a page with a parent) — confirm the menu shows it as
  expandable/collapsible, and that expanding/collapsing actually works.
- [ ] Click a menu item — confirm it navigates to the right page (in Preview/App Player; clicking
  in the Designer canvas itself may just select the widget instead of navigating — check both and
  note the difference).
- [ ] Switch orientation Vertical ↔ Horizontal, confirm the layout visibly changes.
- [ ] **Confirm the old fixed nav bar disappears** once this widget is placed — this is documented,
  intentional behavior; verify it's actually true and doesn't leave a duplicate nav bar.
- [ ] Delete the widget — confirm the old fixed nav bar comes back (or at minimum, that the app
  isn't left with NO navigation at all).

## 4. Usability checklist

- [ ] Is the expand/collapse affordance for nested pages obvious (an arrow/chevron), or hidden?
- [ ] Does the currently-active page get any visual indication in the menu (highlighted/bold), or
  is it impossible to tell where you are?

## 5. Styling/visual checklist

- [ ] Horizontal orientation on Mobile width — does it wrap sensibly, collapse into a hamburger-
  style menu, or overflow/break?
- [ ] Dark theme contrast on menu text/hover states.

## 6. Widget-specific edge cases

- [ ] Test with a page that has `showInMenu` effectively off (if that's a real per-page setting you
  can find in Page Details) — confirm it's correctly excluded from this widget's menu.
- [ ] Add a SECOND Page Navigation widget to the same app in a different section — confirm both
  render correctly and independently (two working nav menus, not one broken).
