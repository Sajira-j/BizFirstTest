# Testing the Video Gallery Widget (`widgetType: 'video-gallery'`)

> Read `../common/resource.md` first (§4 Public/Private assets) and
> `../image-gallery/README.md` first — this widget shares the exact same wizard flow, filter
> fields, zero-config behavior, and security requirements. This doc only covers what's DIFFERENT
> for video.

## 1. What it is

**Label**: Video Gallery · **Description**: "A live, filterable collection of public videos —
Classic Grid or Carousel."

Same live-query mechanism as Image Gallery, filtered to video assets. **Fewer themes than Image
Gallery** — Classic Grid or Carousel only (no Masonry/Ken Burns, which are image-specific visual
treatments that don't make sense for video).

## 2. How to add one

Same wizard flow as Image Gallery (Theme → Size/Automation → land on Configuration), just with the
smaller Grid/Carousel theme choice.

## 3-4-5. Functional / Usability / Styling checklists

Follow `../image-gallery/README.md` §3-5, adjusted for video:
- [ ] Confirm each gallery ITEM is a real playable video (not just a thumbnail image) — click into
  one and confirm playback works.
- [ ] Grid theme: multiple videos visible at once — confirm playing one doesn't force-pause/break
  the others unexpectedly (note actual behavior either way, this is worth knowing rather than
  assuming).
- [ ] Carousel theme: confirm the automation/autoplay setting for ADVANCING SLIDES doesn't also
  force the video itself to autoplay with sound — these are two different "autoplay" concepts
  (slide-advance vs. video-playback), verify they're not conflated.
- [ ] Empty state (no matching public videos) — same check as Image Gallery.

## 6. Widget-specific edge cases

- [ ] Same critical Public/Private security check as Image Gallery — upload a test video marked
  Private, confirm it never appears in a no-filter Video Gallery.
- [ ] Same Document Type/Category real-lookup-field check.
- [ ] If your library has a MIX of video formats (e.g. .mp4 and .webm), confirm the gallery
  correctly plays each one rather than only supporting one format.
