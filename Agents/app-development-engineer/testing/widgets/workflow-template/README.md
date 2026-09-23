# Testing the Workflow Agent Widget (`widgetType: 'workflow-template'`)

> Read `../common/resource.md` first — this widget has known, already-tracked issues listed there
> (§6), please read that before filing new reports.

## 1. What it is

**Label**: Workflow Agent · **Description**: "One AI agent / execution template, with Execute or
Chat Now."

Binds to one real ExecutionTemplate (defined in Flow Studio / WorkDesk) via an
`executionTemplateID`, and renders a card showing that agent's name/description/icon with either an
**Execute** button (one-shot run) or a **Chat Now** button — which one shows depends on the bound
template's own **Type**, not something you configure on the widget itself.

**Chat Now is an intentional full-page redirect, not an inline window.** Clicking it navigates the
whole browser tab to ChatDesk (a separate app) for that ExecutionTemplate — this is by design (a
locked decision, not a bug), not a chat window opening in place. Don't report "Chat Now doesn't
open a chat panel here" as a bug — verify instead that it actually navigates to ChatDesk and that
ChatDesk opens the right agent/template.

## 2. How to add one

**Preferred path — drag from the Toolbox, not the Add Widget modal**: open the Toolbox drawer's
Widgets tab, expand the **"Workflow Agents"** accordion. It lists every enabled ExecutionTemplate BY
NAME. Find the one you want and drag it directly onto a section — this instantly creates a
correctly-configured widget with the right ID, no manual entry needed.

The Add Widget modal's own "Workflow Agent" option also exists (New Widget tab → Widget Type
dropdown) but only offers a **bare numeric ExecutionTemplateID field** — you'd need to already know
the ID. Use the Toolbox path above instead unless you have a specific reason not to.

## 3. Functional test checklist

- [ ] Drag a real agent from the Toolbox "Workflow Agents" list onto a section — confirm it places
  correctly and renders the agent's real name/description/icon (not a placeholder).
- [ ] Click **Execute** (for an Execute-type agent) — confirm real status feedback appears and
  resolves to one of three real outcomes, not silence: (a) the execution completes with no HIL
  input needed — a real "Executed successfully — no input needed" message, (b) the execution
  suspends for input — the real Form/Chat window renders **inline, right there, synchronously**
  (new — see §6), or (c) it fails — a real visible error, not a silent swallow.
- [ ] Click **Chat Now** — confirm it navigates the whole tab to ChatDesk for the right agent (see
  §1 — this is a full-page redirect by design, not an inline window).
- [ ] Click the **Executions** button/link — confirm a real history popover opens showing past runs
  for this template (status, time, error if any) — see §6.
- [ ] Edit an existing Workflow Agent widget — confirm you can rebind it to a different
  ExecutionTemplateID and the change persists.
- [ ] Delete and confirm removal persists.
- [ ] Check in Preview / App Player, not just the Designer canvas — Execute/Chat Now should work
  the same way there.

## 4. Usability checklist

- [ ] Is it clear from the card alone which agent this is and roughly what it does (description
  text)?
- [ ] If Execute fails (e.g. a misconfigured template), is the error message useful, or a raw
  technical error dump?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — the card and its button should stay usable/tappable at every width.
- [ ] Dark theme — check the card's icon/text contrast.

## 6. Widget-specific edge cases — please test these specifically

- [ ] **Execute vs. Chat Now button correctness**: this is driven by the bound ExecutionTemplate's
  **Type** (must be set to a conversational type, e.g. "Conversational") — NOT its Category. If you
  find a template that you believe is meant to be chat-capable but shows "Execute" instead, don't
  assume the widget is broken — that's very likely that specific template's own Type field being
  unset/wrong in Flow Studio/WorkDesk. Note which template, and whether its Type field is visible
  and correctly set in Flow Studio's Edit form (Type/Category fields were recently moved higher up
  in that form specifically because they were easy to miss before — check they're now visible near
  the top, right after Name/Description, without needing to scroll).
- [ ] Test dragging from the Toolbox's "Workflow Agents" list **multiple times in a row, quickly** —
  there was a past report of drag-and-drop from this exact list failing silently in automated
  testing; confirm it works reliably for you doing it by hand, and report if you ever see a drag
  that visibly starts but doesn't result in a placed widget.
- [ ] Test an Execute call that you know/suspect will fail server-side (e.g. a template with
  incomplete configuration) — confirm the failure is visibly surfaced, not silently swallowed.
- [ ] **Executions history (new)**: click the Executions button — confirm a real popover lists past
  runs for this template with status/time/error, not a placeholder. Trigger a fresh Execute, then
  reopen (or watch it auto-open) the Executions popover — confirm the just-triggered run is
  highlighted (e.g. a "Just triggered" badge) so you can see it landed, since there's no standalone
  single-execution detail page in this codebase yet to link to instead.
- [ ] **Inline HIL Form/Chat after Execute (new)**: for a template you know will suspend for input
  (a HIL-capable one), click Execute and confirm the real Form or Chat presentation renders inline
  right on the card — same experience Flow Studio's own execution preview already has — rather than
  nothing happening or only showing up later in an inbox. If you submit through that inline
  form/chat, confirm the reply actually goes through (this uses real authenticated submission — a
  known bug where replies silently failed due to a wrong/unauthenticated URL was found and fixed
  this session, so a regression here would be a high-priority report).
