# Testing the Content Widget (`widgetType: 'content'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Content Widget · **Description**: "Static HTML, Markdown, or plain-text content."

The general-purpose "put some text/rich content on the page" widget. Supports three content
formats and a rich inline HTML editor with an Expression Builder and HTML source editor.

## 2. How to add one

Content Widget is a **safe-default type** — you can either:
- **Drag** it directly from the Toolbox drawer's Widgets tab onto a section (instant-places with
  empty content, no modal), or
- Use **Add Widget → New Widget**, select Content Widget, choose a Content Type
  (HTML/Markdown/Text), optionally type initial content.

Once placed, clicking it opens a **landing view**: the content box IS the editable surface directly
(no "Configuration" click needed first) — HTML gets a real inline rich-text (Tiptap) editor;
Markdown/Text get a plain textarea. An **"Advanced ▾"** link below reveals the fuller
Configuration/Style tabs (Widget Name, Content Type switch, Dynamic Content, Expression Builder).

## 3. Functional test checklist

- [ ] Drag-place from the Toolbox, confirm it instantly appears with an empty, directly-editable
  content box (no modal).
- [ ] Type real content directly into the landing view, click elsewhere, reopen it — confirm your
  edit persisted (this widget saves on dismiss, not via an explicit Save button, in this landing
  view).
- [ ] Click "Advanced ▾", confirm the full Configuration/Style tabs appear, including switching
  Content Type between HTML/Markdown/Text.
- [ ] Test the **Expression Builder** and **HTML Editor** launcher buttons (Content tab) — both
  should open a real, working modal editor, not a blank/broken one.
- [ ] Delete and confirm removal persists after reload.
- [ ] Check the rendered content in Preview / App Player — HTML formatting (bold, links, images
  embedded via the toolbar) should render identically to the Designer canvas.

## 4. Usability checklist

- [ ] Is "click directly into the box to edit" discoverable, or does it feel like you need
  instructions first?
- [ ] Is "Advanced ▾" clearly labeled enough that a user knows more options exist without being
  overwhelming for the common case (someone who just wants to type text)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — long text content should wrap sensibly at every width, no horizontal
  overflow.
- [ ] Check the HTML rich-text toolbar's own controls (alignment, color, image/PDF/video/**audio**
  insert-from-library buttons, table tools) render and are usable in dark theme.

## 6. Widget-specific edge cases

- [ ] Use the toolbar's **"Insert image/PDF/video/audio from library"** buttons (four separate
  icons) — each should open the "Media Library" asset picker. Confirm picking an asset inserts the
  RIGHT embed type for what you actually picked (e.g. clicking the "Insert Video" icon but picking
  an audio file from the library should insert a real `<audio>` player, not mislabeled video markup
  — the embed type is derived from the real file, not which button you clicked).
- [ ] Confirm only **Public** assets are selectable in that picker (see `../common/resource.md`
  §4) — try uploading a Private test asset in digital-assets-library and confirm it does NOT appear
  in this picker.
- [ ] Switch Content Type from HTML to Markdown after already typing HTML content — confirm this
  doesn't silently corrupt/lose your content (note exactly what happens either way).
