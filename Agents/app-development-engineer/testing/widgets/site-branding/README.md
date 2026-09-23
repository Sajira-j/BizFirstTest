# Testing the Site Branding Widget (`widgetType: 'site-branding'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Site Branding · **Description**: "The App's own logo and name, side by side."

Renders the app's own logo (App.IconUrl) and name (App.Name) — pulled from the App's own real
properties (set in App Config), not from anything you configure on the widget itself, except two
optional overrides: **Title override** (defaults to the app's own name) and **Image size** (px,
default 32).

## 2. How to add one

Safe-default type — drag directly from the Toolbox, or Add Widget with optional overrides.

## 3. Functional test checklist

- [ ] Drag-place with no overrides, confirm it shows the app's REAL name and logo (set/verify these
  in App Config → General first if the app doesn't have a logo set — test with an app that has a
  real logo image, not a blank default).
- [ ] Set a Title override, confirm it now shows the override text instead of the app's real name
  (and that the app's actual name elsewhere, e.g. the Designer's own toolbar, is unaffected — this
  is a display-only override, not a rename).
- [ ] Change the app's real name in App Config, confirm this widget updates to match (when no
  override is set) after a reload.
- [ ] Try Image sizes at the extremes (very small, e.g. 8px, and very large, e.g. 200px) — confirm
  the logo scales without breaking layout or becoming illegibly tiny/absurdly huge in a way that
  looks broken rather than intentional.
- [ ] Delete and confirm removal persists.

## 4. Usability checklist

- [ ] Is "Title override" clearly explained as optional with a sensible placeholder/default hint
  (not looking like a required field)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — logo+name shouldn't overlap or get squeezed illegibly at narrow
  widths.
- [ ] Test with an app that has NO logo set (`IconUrl` empty) — confirm there's a sensible fallback
  (not a broken image icon).

## 6. Widget-specific edge cases

- [ ] Test with a very long app name (or a long Title override) — confirm it truncates/wraps
  sensibly rather than breaking the surrounding layout.
