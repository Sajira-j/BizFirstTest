# Testing the Form Widget (`widgetType: 'form'`)

> Read `../common/resource.md` first — environment setup, DevTools usage, Public/Private asset
> rules, and the bug-report template all live there.

## 1. What it is

**Label**: Form Widget
**Registry description**: "Create, edit, view, or list records from an Atlas Forms form."

Binds to one form defined in Atlas Forms (a separate, shared form-building system used across
BizFirst) via a `formId`, and renders it in one of four modes: **List**, **Edit**, **View**, or
**Create**. This is the "any table, generic CRUD" building block of App Studio — one Form Widget
placement = one way of interacting with one Atlas Forms-defined form/table.

## 2. How to add one

1. Open App Studio Designer, select or create an app, click into a section.
2. Click **Add Widget** → **New Widget** tab.
3. In the **Widget Type** dropdown, select **Form Widget** (this is the default-selected type when
   the dropdown first opens).
4. Fields:
   - **Widget name** (required) — internal name, shown in Section Details' widget list.
   - **Form** (required) — a searchable lookup field. Type to search by form name, select from the
     dropdown. This is a real search-as-you-type field, not a bare ID — if you only see a plain
     number box here, that's a regression, report it.
   - **Render Mode** (required, defaults to List) — dropdown: List / Edit / View / Create.
5. Click **Add Widget**.

This type is **not** in the "safe default" drag-instant-place set — it always goes through the Add
Widget modal above (a form binding is a hard-required field with no sensible default).

## 3. Functional test checklist

- [ ] Create with a valid form selected in each of the 4 render modes — confirm each mode actually
  renders differently (List = a records table/grid; Edit/View/Create = a single-record form).
- [ ] Try to click **Add Widget** with no form selected — confirm you get a clear validation
  message ("Select a form"), not a silent failure or a crash.
- [ ] Edit an existing Form Widget (click it on canvas → Widget Details) — confirm you can change
  the bound form and the render mode, save, and the change actually persists after a page reload.
- [ ] Delete a Form Widget — confirm it's gone from the section's widget list, and that it doesn't
  reappear after reload.
- [ ] Check the same widget in **Preview** and in the real **App Player** tab (see
  `../common/resource.md` §3) — confirm List/Edit/View/Create modes all render correctly outside
  the Designer canvas too, not just inside it.
- [ ] In **List** mode, if the bound form has any records, confirm they actually show up (not just
  an empty table with no data-loading indication either way).
- [ ] In **Create** mode, actually submit a new record through the rendered form and confirm it's
  saved (check it shows up if you then view the same form in List mode).

## 4. Usability checklist

- [ ] Is it obvious from the "Render Mode" label alone what each of the 4 options does, or would a
  first-time user need to guess/experiment?
- [ ] Does the Form search-lookup field give reasonable results quickly, or does it feel slow /
  show too many irrelevant results?
- [ ] Is there a loading indicator while the form's fields are being resolved, or does the widget
  area just sit blank for a moment with no feedback?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile preview toggle (canvas top-left) — does the rendered form's field
  layout adapt sensibly, or do fields overflow/get cut off on Mobile?
- [ ] Dark theme — check text contrast is readable on every field type this form uses (text inputs,
  dropdowns, checkboxes, etc.) — Atlas Forms controls are a separate design system from App Studio
  itself, so don't assume they automatically match; report anything that looks visually "foreign"
  to the rest of the page.

## 6. Widget-specific edge cases

- [ ] **The "Configure Form" icon**: in Widget Details (edit view) for a Form Widget, next to the
  **Form** field there should be a small settings/gear-style icon. Click it — it should open Atlas
  Forms' own reusable Form Editor **inline, in a modal**, letting you edit the form's actual field
  definitions without leaving App Studio Designer. Verify: (a) the icon is there at all, (b) it
  opens a real, functional form editor (not a blank/broken modal), (c) a change you make there and
  save is reflected when you reopen the widget's List/View render afterward. (This gear-next-to-
  lookup pairing is now the reference pattern reused across Atlas Forms' other form-pickers too —
  if it changes here, check it didn't regress there as well.)
- [ ] Try binding the SAME form to two different Form Widgets in different sections, one in List
  mode and one in Create mode — confirm they behave independently (creating a record via one
  doesn't break the other's list, etc.).
- [ ] Try an "Existing Widget" placement (Add Widget → **Existing Widget** tab) — pick a Form Widget
  that already exists elsewhere in the app and place a second copy of it in a new section. Confirm
  both placements work and share the same underlying Widget definition (editing the form binding on
  one — does it affect the other? Note what you observe either way, this is worth knowing).
