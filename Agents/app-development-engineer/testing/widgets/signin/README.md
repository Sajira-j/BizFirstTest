# Testing the Sign In / Sign Out Widget (`widgetType: 'signin'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Sign In / Sign Out · **Description**: "Sign-in link when signed out; a user menu with
sign-out when signed in."

A thin wrapper around shared BizFirst auth UI. One config field: an optional custom **Sign-in URL**
(defaults to the central login app if left blank).

## 2. How to add one

Safe-default type — drag directly from the Toolbox, or Add Widget with an optional Sign-in URL
override.

## 3. Functional test checklist

- [ ] Drag-place with default (blank) Sign-in URL — in App Player, open in an **incognito/private
  window** (so you're signed out) and confirm it shows a real sign-in link that goes to the correct
  central login app.
- [ ] While signed IN (normal window), confirm it instead shows a user menu with your real
  name/avatar and a working Sign Out option.
- [ ] Click Sign Out — confirm it actually signs you out (you're redirected/shown as signed-out
  state, and a protected action elsewhere in the app now requires signing in again).
- [ ] Set a custom Sign-in URL, confirm the signed-out state's link now points there instead of the
  default.
- [ ] Delete and confirm removal persists.

## 4. Usability checklist

- [ ] Is the signed-in user menu's content clear (name shown correctly, obvious how to sign out)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — user menu dropdown shouldn't overflow the viewport or get clipped on
  narrow widths.
- [ ] Dark theme contrast on the avatar/name/dropdown.

## 6. Widget-specific edge cases

- [ ] Test this widget on a page alongside a **Notifications** or **HIL Inbox** widget — confirm
  they all agree on the same signed-in/signed-out state (no widget showing you as signed in while
  another acts as if you're signed out).
- [ ] Leave the custom Sign-in URL field, save, then clear it back to blank — confirm it correctly
  falls back to the default central login app again (not stuck on the old custom value, and not
  broken).
