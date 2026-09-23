# Testing the HIL Inbox Widget (`widgetType: 'hil-inbox'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: HIL Inbox · **Description**: "A human-in-the-loop actionable inbox — approvals, forms,
and tasks routed to this app's users."

A thin wrapper around the shared BizFirst HIL (Human-In-the-Loop) UI package — shows the signed-in
user's own actionable inbox items (approvals/forms/tasks routed to them), scoped automatically from
their tenant/auth context. **No configuration fields at all** — it's zero-config by design.

## 2. How to add one

Safe-default type — drag directly from the Toolbox Widgets tab, instant-places with no config
needed. (The Add Widget modal's version, if used instead, shows an informational "No configuration
needed" message rather than any fields.)

## 3. Functional test checklist

- [ ] Drag-place, confirm it renders without any config step.
- [ ] **Important**: this widget will look empty/trivial with no test data — to see meaningful
  content you need real HIL items routed to your signed-in account. Ask your team lead how to
  generate a test approval/task if none exist for you already (this may require triggering a real
  workflow elsewhere in the system that creates a HIL session).
- [ ] With at least one real inbox item: confirm it shows up with correct details, and that acting
  on it (approve/reject/fill a form, whatever the item type supports) actually works and the item
  updates/disappears from the inbox afterward.
- [ ] Delete the widget, confirm removal persists.
- [ ] Check in Preview / App Player — confirm it still correctly scopes to the SAME signed-in user
  (not a different/wrong user's inbox).

## 4. Usability checklist

- [ ] With zero inbox items, is the empty state clear ("No pending items" or similar), or does it
  look broken/blank?
- [ ] Is it clear what action each item type expects from the user (approve vs. fill a form vs.
  something else)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — item list should remain readable/tappable at every width.
- [ ] Dark theme contrast, especially any status badges/colors this widget uses (pending/approved/
  rejected states, if applicable).

## 6. Widget-specific edge cases

- [ ] Confirm this widget genuinely shows ONLY items for the currently signed-in user — if you have
  access to a second test account, sign in as that account and confirm the inbox contents differ
  appropriately (no cross-user data leak).
- [ ] Since this is zero-config, there's nothing to misconfigure — focus testing effort on real
  data correctness and the action-taking flow rather than the (nonexistent) config screen.
