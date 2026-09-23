# Testing the PDF Widget (`widgetType: 'pdf'`)

> Read `../common/resource.md` first — especially §4 (Public/Private assets).
> The **Media Source picker** works identically to the Image widget — see `../image/README.md` §6.

## 1. What it is

**Label**: PDF · **Description**: "A single PDF — URL, optional title, shown as a link or an
inline embed."

## 2. How to add one

Add Widget → New Widget → **PDF**. Fields:
- **PDF URL** (required, Media Source picker)
- **Title** (optional) — link text / accessible name.
- **Display Mode** (Link / Embed, default Link):
  - **Link**: a styled button/link that opens the PDF in a new tab.
  - **Embed**: an inline PDF viewer directly on the page (fixed height).

## 3. Functional test checklist

- [ ] Create in **Link** mode with a real PDF, confirm a real, clickable, correctly-styled link/
  button renders (check via DevTools that it isn't just plain unstyled text pretending to be a
  link).
- [ ] Click the Link-mode button in App Player — confirm it opens the actual PDF in a new tab, and
  the PDF genuinely loads (not a 404/broken link).
- [ ] Create in **Embed** mode with the same PDF — confirm an inline viewer actually renders the PDF
  content on the page (not a blank box or a broken-plugin icon — some browsers require a PDF viewer
  plugin/extension; note your browser and what you saw if it doesn't render).
- [ ] Try Add Widget with no PDF URL — confirm validation error, no crash.
- [ ] Edit, change fields (including switching Link ↔ Embed), save, reload, confirm persistence.
- [ ] Delete, confirm removal persists.
- [ ] Check in Preview / App Player.

## 4. Usability checklist

- [ ] Is it clear from the rendered widget alone whether clicking it opens a new tab (Link mode) vs.
  is already showing the content inline (Embed mode) — i.e., does the UI set the right expectation?
- [ ] With Title left blank, does the Link-mode button show a sensible fallback label (not literally
  blank/unclickable-looking)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — Embed mode's fixed height: does it stay usable/scrollable on Mobile,
  or does a tall fixed-height PDF viewer dominate/break the mobile layout? This is worth specific
  attention since PDF embeds are a known "wide desktop content, awkward on mobile" pattern.
- [ ] Link mode's button styling — dark theme contrast/consistency with other buttons on the page.

## 6. Widget-specific edge cases

- [ ] Same Private-asset security check as the other media widgets — confirm a Private PDF never
  appears in the "Browse Media Library" picker, and additionally: if you have a Private PDF's raw
  URL, try pasting it directly into "Enter Custom URL" — confirm the widget still can't actually
  load/display it in App Player as an anonymous visitor (Private assets require an authenticated,
  expiring fetch — this should fail gracefully for a signed-out visitor, not silently "work" in a
  way that defeats the Public/Private distinction).
- [ ] Test a genuinely large PDF (many pages/MB) in Embed mode — confirm it doesn't hang the page or
  take an unreasonably long time to become interactive.
