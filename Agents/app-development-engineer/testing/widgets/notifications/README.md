# Testing the Notifications Widget (`widgetType: 'notifications'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Notifications · **Description**: "A notifications bell with unread count and dropdown
list."

A thin wrapper around the shared BizFirst notifications UI. One config field: optional **View-all
URL** (omits the "view all" link entirely if left blank).

## 2. How to add one

Safe-default type — drag directly from the Toolbox, or Add Widget with an optional View-all URL.

## 3. Functional test checklist

- [ ] Drag-place, confirm a bell icon renders. If you have zero real notifications, ask your team
  lead how to trigger one for your test account (similar to HIL Inbox, this needs real backend
  activity to show meaningful content).
- [ ] With at least one real notification: confirm the unread count badge shows the right number,
  clicking the bell opens a dropdown listing it correctly, and clicking the notification itself does
  something sensible (navigates/marks read).
- [ ] Mark a notification read — confirm the unread count decrements correctly.
- [ ] Set a View-all URL, confirm a "view all" link appears in the dropdown and goes to the right
  place; leave it blank, confirm the link is correctly omitted (not shown as a dead/broken link).
- [ ] Delete and confirm removal persists.

## 4. Usability checklist

- [ ] Is the unread count badge clearly visible against the bell icon in both light-content and
  dark-content page backgrounds?
- [ ] With zero notifications, does the dropdown show a sensible empty state?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — dropdown positioning shouldn't run off-screen on narrow widths.
- [ ] Dark theme contrast on the badge/dropdown.

## 6. Widget-specific edge cases

- [ ] Test with a LARGE unread count (10+, if you can generate that many test notifications) —
  confirm the badge handles it sensibly (e.g. "9+" style truncation) rather than an unreadably wide
  number.
- [ ] Same cross-widget signed-in-state consistency check as the Sign In widget's README §6 —
  Notifications should only ever show YOUR OWN notifications, never another user's.
