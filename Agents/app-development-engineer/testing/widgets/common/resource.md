# App Studio Widget Testing — Shared Resource Guide

Read this once before testing any widget type. Every per-widget `README.md` under
`widgets/{widgetType}/` assumes you've read this and links back here instead of repeating it.

---

## 1. Environment setup

You need three things running locally:

### 1.1 App Studio Designer (the app you're testing)
- URL: `http://localhost:6109`
- This is a Vite dev app. If it isn't already running, ask whoever set up your machine for the
  start command for `app-studio/apps/app-studio-designer` (typically `npm run dev` / `pnpm dev`
  from that folder). **Do not stop/restart it yourself if someone else is actively using it** —
  check with your team lead first.

### 1.2 Consolidated WebApi (the backend)
- URL: `https://localhost:10001` (Swagger UI at `https://localhost:10001/swagger/index.html` — a
  good first check that it's actually up).
- Real start command (run from
  `C:\BizFirstGO_FI_AI\BizFirstPayrollV3\src\mvc-server\Solutions\AiUltimate\BizFirst.Ai.Consolidated.WebApi`):
  ```
  dotnet run --project "BizFirst.Ai.Consolidated.WebApi.csproj" --launch-profile "BizFirst.Ai.Consolidated.WebApi" -m:2
  ```
  (`-m:2` caps build parallelism — keep it, it's there on purpose for this machine.)
- If a widget you're testing looks broken with no visible content anywhere, **check this first** —
  a very common failure mode is simply "the backend isn't running," which looks like a dozen
  different bugs depending on the widget.
- If `https://localhost:10001/swagger/index.html` doesn't load: the server may be down or still
  starting. Give it 15-30 seconds after starting before assuming it's broken.

### 1.3 digital-assets-library (only needed for media widgets)
- URL: `http://localhost:6111`
- Only needed if you're testing Image / Video / Audio / PDF / any of the four Gallery widgets —
  this is where you upload test files and mark them Public or Private (see §4 below, this
  Public/Private distinction is a real security boundary you're expected to test).

### 1.4 Signing in
- Navigate to `http://localhost:6109`, you'll be redirected to a login page if you don't have an
  active session. Sign in with your own real account — there are no shared test credentials for
  this. If you don't have an account yet, ask your team lead.
- Sessions expire periodically. If a page you were just using suddenly shows "not rendering" or a
  blank screen, check whether you got redirected to the login page in a tab you didn't notice.

---

## 2. Using Chrome DevTools for end-to-end testing

There's no special extension required — Chrome's own built-in DevTools is the real, always-available
tool for this. Open it with **F12** or **Ctrl+Shift+I** (Windows) while on the App Studio Designer
tab. The four tabs you'll use for every widget test:

### Console tab
- Watch this while you create/edit/delete a widget. A widget that "looks fine" but is silently
  throwing a JS error in the console is still a bug — report it.
- Red text = error, yellow = warning. Not every yellow warning is a real bug (this app has some
  known, harmless third-party warnings), but flag anything you're not sure about.

### Network tab
- Filter by `Fetch/XHR` to see just the API calls, not images/scripts/etc.
- When you save a widget's configuration, you should see a request to something like
  `/api/v1/app-studio/widgets` or `/api/v1/app-studio/app-widgets/{id}` — click it, check the
  **Status** column (200/201 = success; 4xx/5xx = a real failure you should report, even if the UI
  didn't show an obvious error) and inspect the **Response** tab for what actually came back.
- This is also how you catch the "silent failure" class of bug — a button that appears to work but
  the network request actually failed.

### Elements tab
- Right-click any part of a widget on the canvas → **Inspect** to see its real rendered HTML/CSS.
  Useful for styling checks — e.g. confirming an image is really using `object-fit: cover` as
  configured, or checking why something looks misaligned.

### Device toolbar (responsive testing)
- Click the phone/tablet icon in DevTools (top-left of the DevTools panel, or **Ctrl+Shift+M**) to
  simulate mobile/tablet viewport sizes. **Note**: App Studio Designer's own canvas already has a
  built-in Desktop/Tablet/Mobile preview toggle (top-left of the canvas area, above the page) — use
  THAT one for the per-widget "styling checklist" responsive checks in each widget's README, since
  it reflects this app's own real breakpoint behavior. Chrome's device toolbar is a good second
  check / cross-reference, not a replacement.

---

## 3. Where widgets actually render — Designer canvas vs. Preview vs. App Player

A widget can look different (or work differently) in each of these three places. **Every widget's
functional checklist asks you to check more than one of these** — don't stop at the Designer
canvas:

1. **Designer canvas** (`localhost:6109`, editing an app) — a design-time render, sometimes with
   studio-mode overlays (selection outlines, hover labels) that aren't part of the real page.
2. **Preview panel** — if enabled (View menu → "Enable Preview Panel"), shows a closer-to-real
   render inline in the Designer.
3. **App Player** (`localhost:6130`) — the REAL end-user render, no design-time overlays at all.
   Reached via the **Preview** button in the top toolbar (opens a new tab). This is the one that
   matters most — if a widget looks right in the Designer canvas but wrong here, that's a real bug.

---

## 4. Public vs. Private assets — a real security boundary, not just a setting

When you upload a file in digital-assets-library, you choose **Public** or **Private**. This isn't
cosmetic:

- **Public** assets are meant to be safely embeddable in a real, published, anonymous-visitor-facing
  page (App Player). Their URL is meant to work without the visitor being logged in at all.
- **Private** assets require an authenticated, expiring, presigned fetch — safe for the internal
  digital-assets-library UI (you're logged in there), but if a private asset's raw URL ever got
  embedded into a real published page, a site visitor's browser would fail to load it (or worse,
  if something's misconfigured, could leak content that shouldn't be public).

**This is why every media widget's edge-case checklist asks you to specifically verify a Private
asset never shows up** in an Image/Video/Audio/PDF Gallery widget, and that the "Browse Media
Library" picker for the single Image/Video/Audio/PDF widgets doesn't let you pick a private asset
either. If you ever see a private asset show up somewhere it shouldn't, **stop and report it
immediately as a high-priority finding** — this is a real data-exposure class of bug, not a
cosmetic one.

---

## 5. How to report a finding

Use this exact template for every bug/usability issue/missing-functionality/styling issue you find,
so everyone's reports are comparable:

```
### [Widget Type] — [short title]

**Category**: Bug / Usability / Missing functionality / Styling / Security
**Severity**: High / Medium / Low  (High = data exposure, crash, or a core action completely
                                     broken; Medium = works but wrong/confusing; Low = cosmetic)

**What I did (repro steps)**:
1. ...
2. ...
3. ...

**What happened**:
...

**What I expected instead**:
...

**Where** (check all that apply): [ ] Designer canvas  [ ] Preview panel  [ ] App Player
**Screenshot**: [attach or link]
**Console/Network errors** (if any): [paste relevant lines]
```

Save your findings as a markdown file per widget type, or one combined file — ask your team lead
which they'd prefer for how this gets consolidated.

---

## 6. Known, already-tracked issues (don't re-report these — but DO verify they're still true)

A few things were already found and are being tracked/fixed as of this document's writing. If you
hit these, you don't need to file a new report — but please still note in your results whether you
saw the same behavior, since that's useful confirmation:

- **Workflow Agent widget**: clicking Execute/Chat Now doesn't always give clear visual
  success/failure feedback for every path.
- **Workflow Agent widget, chat-type templates**: whether a template shows "Chat Now" vs. "Execute"
  depends on that template's **Type** being correctly set to a conversational type in Flow
  Studio/WorkDesk — not its Category. If a template you'd expect to be chat-capable shows "Execute"
  instead, that's very likely a data issue with that specific template (its Type field), not a
  widget bug — flag which specific template it is rather than filing it as "the widget is broken."
