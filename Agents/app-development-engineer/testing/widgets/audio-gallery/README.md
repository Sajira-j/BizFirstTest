# Testing the Audio Gallery Widget (`widgetType: 'audio-gallery'`)

> Read `../common/resource.md` first (§4 Public/Private assets) and
> `../image-gallery/README.md` first — same wizard flow, filter fields, zero-config behavior, and
> security requirements. This doc only covers what's DIFFERENT for audio.

## 1. What it is

**Label**: Audio Gallery · **Description**: "A live, filterable collection of public audio tracks —
Classic Grid or Carousel."

Same live-query mechanism, filtered to audio assets. Grid or Carousel themes only. **Note**: the
default column count for this gallery type is 2 (vs. 3 for Image/Video/PDF galleries) — this is a
deliberate, sensible adjustment since audio players are visually narrower items; confirm this
default is actually what you see when you create a fresh one.

## 2. How to add one

Same wizard flow as Image Gallery.

## 3-4-5. Functional / Usability / Styling checklists

Follow `../image-gallery/README.md` §3-5, adjusted for audio:
- [ ] Confirm each gallery item is a real, individually playable audio track (title/label per item
  if the underlying asset has a name — check items are distinguishable from each other, not just a
  row of identical-looking unlabeled players).
- [ ] Given audio players are visually small (see `../audio/README.md` §4 for the same narrow-player
  concern on the single Audio widget), pay close attention to whether each item is comfortably
  sized/usable in Grid mode, not squeezed too small at the default 2-column width.
- [ ] Carousel theme: confirm playing one track and then advancing to the next (via automation or
  manually) stops/pauses the previous one sensibly, rather than overlapping audio playback.

## 6. Widget-specific edge cases

- [ ] Same critical Public/Private security check — upload a test audio file marked Private, confirm
  it never appears in a no-filter Audio Gallery.
- [ ] Same Document Type/Category real-lookup-field check.
