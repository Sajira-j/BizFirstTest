# Testing the Image Widget (`widgetType: 'image'`)

> Read `../common/resource.md` first — especially §4 (Public/Private assets) before testing.

## 1. What it is

**Label**: Image · **Description**: "A single photo — URL, alt text, optional caption, object-fit,
and an optional link."

Renders one image with real accessibility (alt text) and layout controls, optionally wrapped in a
link.

## 2. How to add one

Add Widget → New Widget → **Image**. Fields:
- **Image URL** (required) — see §6 below, this is a special two-mode field, not a plain text box.
- **Alt Text** (recommended, not hard-required) — auto-filled from the picked asset's name if you
  used the Media Library path and left this blank.
- **Caption** (optional) — shown below the image if set.
- **Object Fit** (Cover / Contain / Fill, default Cover) — standard CSS `object-fit` behavior.
- **Link URL** (optional) — wraps the image in a link if set.
- **Presentation Style** (if present in this build — Plain / Framed / Polaroid / Ken-Burns-on-hover)
  — a lighter visual-treatment option added alongside the Gallery widgets' theme system. Confirm
  this field actually exists in the Configuration screen; if it's missing, that's worth noting
  (recently added, may not be in every build yet).

Not a safe-default type — always via Add Widget, since Image URL is hard-required.

## 3. Functional test checklist

- [ ] Create with a real image, confirm it renders (check `naturalWidth`/`naturalHeight` &gt; 0 via
  DevTools Elements/Console — a broken image can visually "look present" as a placeholder icon at
  first glance, don't rely on eyeballing alone).
- [ ] Try Add Widget with NO Image URL set — confirm a clear validation error ("Enter an Image
  URL"), not a silent failure or crash.
- [ ] Edit an existing Image widget, change the URL/alt text/caption/object-fit/link, save, reload,
  confirm all changes persisted.
- [ ] Delete and confirm removal persists.
- [ ] Check in Preview / App Player.
- [ ] Test the Link URL — confirm clicking the rendered image (App Player, not Designer canvas)
  actually navigates there.

## 4. Usability checklist

- [ ] Is Alt Text's purpose clear (accessibility/SEO), or does it read as just another random field?
- [ ] Does Object Fit's effect make visual sense when you switch between Cover/Contain/Fill on a
  non-square test image (pick one that's clearly not square so the difference is obvious)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — confirm the image scales/crops sensibly at every width per its Object
  Fit setting.
- [ ] Test Caption rendering — does it look intentional (styled text below the image), not just
  raw unstyled text?
- [ ] If Presentation Style exists (§2 above), test each option and confirm they look visually
  distinct from each other, not just cosmetic label changes on identical rendering.

## 6. Widget-specific edge cases — the Media Source picker

The **Image URL** field is actually a two-mode component:
- If empty, you first see a choice: **"Browse Media Library"** or **"Enter Custom URL"**.
- **Browse Media Library** opens the same asset picker used elsewhere — pick a real uploaded asset,
  its URL fills in automatically, and (if Alt Text was empty) Alt Text auto-fills from the asset's
  name.
- **Enter Custom URL** lets you type/paste any URL directly.
- Once a value exists (either way), the chooser is replaced by a plain text field with a small
  folder/library icon next to it — clicking that icon reopens the picker to change the source later.

Test:
- [ ] Both paths (Browse Media Library AND Enter Custom URL) actually work and produce a rendering
  image.
- [ ] After picking via Library, click the small library icon again and pick a DIFFERENT asset —
  confirm it correctly replaces the previous one (not appending/duplicating).
- [ ] **Security check**: upload a test image marked **Private** in digital-assets-library, then try
  to find it in this widget's "Browse Media Library" picker — confirm it does NOT appear (only
  Public assets should ever be selectable here). If a private asset DOES show up, this is a
  high-priority security finding — report it immediately per `../common/resource.md` §4/§5.
- [ ] This same picker component is reused in this widget's edit screen too — confirm "Browse Media
  Library" is available when EDITING an existing Image widget, not just when creating one.

## 7. Known limitation — full-resolution images, no thumbnails

The backend serves the **original uploaded file** for every image request — there is no
server-side thumbnail/resize capability anywhere in the pipeline. A large original photo used here
downloads in full even at a small display size (the widget adds `loading="lazy"` so at least
offscreen images defer loading, but that doesn't reduce the bytes transferred). This is a known,
accepted gap, not a bug to report — but if you're testing perceived load performance, be aware a
"slow" image is likely just a large original file, not a rendering problem.
