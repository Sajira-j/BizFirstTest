# Testing the PDF Gallery Widget (`widgetType: 'pdf-gallery'`)

> Read `../common/resource.md` first (§4 Public/Private assets) and
> `../image-gallery/README.md` first — same wizard flow, filter fields, zero-config behavior, and
> security requirements. This doc only covers what's DIFFERENT for PDF.

## 1. What it is

**Label**: PDF Gallery · **Description**: "A live, filterable collection of public PDF documents —
Classic Grid or Carousel."

Same live-query mechanism, filtered to PDF assets. Grid or Carousel themes only.

## 2. How to add one

Same wizard flow as Image Gallery. One extra note: the Document Type/Document Category filters here
should render as real searchable dropdowns (confirmed at build time to be an intentional
consistency improvement over some other config screens' plain fields) — double-check they ARE
dropdowns here specifically, and report if you find these reverted to plain text/number fields.

**New: Display Mode.** The Configuration screen now has a **Display Mode** field: **Link** (each
PDF is a clickable link/thumbnail that opens the file — original behavior) or **Embed** (each PDF
renders inline via a real `<embed>`, showing actual page content directly in the gallery). Embed
mode is allowed in both **Grid** and **Carousel** layouts — in Grid, each cell shows its own
embedded viewer; in Carousel, one full-size viewer shows at a time. Confirm the field exists, and
that switching to Embed actually shows real PDF page content inline (not just a bigger link/icon).

## 3-4-5. Functional / Usability / Styling checklists

Follow `../image-gallery/README.md` §3-5, adjusted for PDF:
- [ ] Confirm each gallery item is either a real clickable link to a PDF (opens/downloads correctly)
  or, if this gallery's items render as thumbnails, that clicking one does something sensible (opens
  the PDF).
- [ ] With PDFs of very different page counts/sizes, confirm the gallery item representation (title/
  icon/thumbnail — whatever this widget actually shows per item) stays consistent and doesn't break
  for an unusually large file.

## 6. Widget-specific edge cases

- [ ] Test **Display Mode = Embed** in both Grid and Carousel — confirm real page content renders
  inline, not a link or broken plugin placeholder (some browsers render PDF embeds differently —
  note which browser you tested in if you see anything odd).
- [ ] Same critical Public/Private security check — upload a test PDF marked Private, confirm it
  never appears in a no-filter PDF Gallery.
- [ ] Confirm clicking a PDF item in App Player (not just the Designer canvas) actually opens/loads
  the real file, not a broken link.
