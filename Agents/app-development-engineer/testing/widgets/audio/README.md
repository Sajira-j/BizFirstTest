# Testing the Audio Widget (`widgetType: 'audio'`)

> Read `../common/resource.md` first — especially §4 (Public/Private assets).
> The **Media Source picker** works identically to the Image widget — see `../image/README.md` §6.

## 1. What it is

**Label**: Audio · **Description**: "A single audio player — URL, optional title/caption,
autoplay/loop."

## 2. How to add one

Add Widget → New Widget → **Audio**. Fields:
- **Audio URL** (required, Media Source picker)
- **Title** (optional) — shown above the player as a label (audio has no visual thumbnail the way
  video has a poster, so Title is its equivalent "what am I listening to" identifier — check this
  actually renders visibly, it's easy for a label like this to get missed in styling).
- **Caption** (optional)
- **Autoplay** (default OFF)
- **Loop** (default OFF)

## 3. Functional test checklist

- [ ] Create with a real audio file, confirm a working `<audio>` player renders and actually plays
  sound when you press play.
- [ ] Try Add Widget with no Audio URL — confirm validation error, no crash.
- [ ] Edit, change fields, save, reload, confirm persistence.
- [ ] Delete, confirm removal persists.
- [ ] Check in Preview / App Player.

## 4. Usability checklist

- [ ] **This is the one to pay closest attention to for layout**: an audio player has a very small,
  narrow native UI (just a thin control bar, no visual thumbnail). In a wide or flex-row layout
  context, does it render at a reasonable, visible size, or does it get squeezed down to a tiny
  sliver that's hard to notice/use? (There's a known past report of exactly this happening in a
  narrow header-row context — check whether it's still an issue in whatever section layout you test
  in, and report if the player is ever rendered unreasonably small.)

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — same narrow-player-width concern as above, check at all three sizes.
- [ ] Confirm the Title label (if set) is visually distinct from the player itself (not overlapping
  or crammed against it).

## 6. Widget-specific edge cases

- [ ] Same Autoplay-policy and Loop checks as the Video widget (§6 there) — audio autoplay is
  subject to the same browser restrictions.
- [ ] Same Private-asset security check — confirm a Private audio file never appears in the "Browse
  Media Library" picker.
