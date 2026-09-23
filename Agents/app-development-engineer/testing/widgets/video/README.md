# Testing the Video Widget (`widgetType: 'video'`)

> Read `../common/resource.md` first — especially §4 (Public/Private assets).
> The **Media Source picker** (Browse Media Library vs. Enter Custom URL) works identically here to
> the Image widget — see `../image/README.md` §6 for the full explanation of that field; this doc
> won't repeat it.

## 1. What it is

**Label**: Video · **Description**: "A single video player — URL, optional poster/caption,
autoplay/loop/controls."

## 2. How to add one

Add Widget → New Widget → **Video**. Fields:
- **Video URL** (required, Media Source picker — see Image widget's README §6)
- **Poster URL** (optional) — thumbnail shown before play.
- **Caption** (optional)
- **Autoplay** (checkbox, default OFF — deliberately never defaults on)
- **Loop** (checkbox, default OFF)
- **Controls** (checkbox, default ON)

## 3. Functional test checklist

- [ ] Create with a real video file, confirm it actually renders a working `<video>` player (check
  via DevTools Elements that `readyState`/`duration` are real values, not just that a player-shaped
  box appears — an unloadable video URL can still show player CONTROLS with no actual video).
- [ ] Try Add Widget with no Video URL — confirm proper validation error, no crash.
- [ ] Test actually pressing Play — confirm real playback with audio/video in sync.
- [ ] Edit, change fields, save, reload, confirm persistence.
- [ ] Delete, confirm removal persists.
- [ ] Check in Preview / App Player.

## 4. Usability checklist

- [ ] Is Autoplay clearly labeled as off-by-default (a real UX best practice — confirm it stays that
  way and nothing accidentally autoplays with sound on page load)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — player should resize sensibly, controls remain tappable on Mobile.
- [ ] Test with a Poster URL set — confirm it shows before play; without one, confirm there's a
  sensible fallback (not a jarring black box).

## 6. Widget-specific edge cases

- [ ] Turn Autoplay ON — confirm it respects browser autoplay policies sensibly (most browsers
  require muted autoplay; verify it doesn't error out or fail silently, and check whether it's
  actually muted when autoplaying, which is the correct/expected behavior for autoplay to work at
  all in modern browsers).
- [ ] Turn Loop ON, let a short video play to the end, confirm it actually loops instead of
  stopping.
- [ ] Turn Controls OFF — confirm the player still renders but with no visible play/pause/seek bar
  (useful for background-video use cases) — if Autoplay is also off in this state, make sure there's
  still SOME way to start playback, or note that this combination effectively makes the video
  unplayable by a normal user (worth flagging as a usability concern if so).
- [ ] Same Private-asset security check as the Image widget (§6 there) — confirm a Private video
  never appears in the "Browse Media Library" picker.
