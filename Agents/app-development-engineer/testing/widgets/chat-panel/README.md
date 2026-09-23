# Testing the Chat Panel Widget (`widgetType: 'chat-panel'`)

> Read `../common/resource.md` first.

## 1. What it is

**Label**: Chat Panel · **Description**: "An embedded chat window for triggering and conversing
with one process — new conversations only; resuming a past one is read-only for now."

Embeds a full ChatDesk-style chat window directly on the page, bound to one process
(`processID`, optionally an `executionTemplateID` to pre-hydrate the first message). Unlike the
Workflow Agent widget's "Chat Now" button (which opens a chat overlay on click), this widget IS the
chat window, always visible.

## 2. How to add one

Add Widget → New Widget → select **Chat Panel**. Fields:
- **ProcessID** (required)
- **ExecutionTemplateID** (optional — hydrates the first message's input data)
- **Welcome title** / **Welcome subtitle** (optional — shown before the first message; if left
  blank, the chat window's own built-in defaults apply)
- **"Show past-conversations panel"** checkbox (optional)

Not a safe-default type — always goes through Add Widget with a required ProcessID.

## 3. Functional test checklist

- [ ] Create with a valid ProcessID — confirm a real, functional chat window renders (not a blank
  box or an error).
- [ ] Send a message in the embedded chat — confirm you get a real response (this is a live
  conversation, not a mock).
- [ ] Toggle "Show past-conversations panel" on — confirm a conversation list actually appears
  alongside the chat.
- [ ] **Known, accepted limitation** (not a bug to report): resuming a PAST conversation shows
  read-only history with a disabled composer — new conversations work fully. Confirm this matches
  what you see; if a past conversation instead crashes or shows something worse than "read-only,"
  that IS worth reporting.
- [ ] Delete and confirm removal persists.
- [ ] Check in Preview / App Player — confirm the chat window is fully functional there too, not
  just in the Designer.

## 4. Usability checklist

- [ ] Is the welcome title/subtitle (if set) actually shown before the first message, in a way
  that reads naturally?
- [ ] Is it clear to an end user that they can type and send a message (obvious input box,
  send affordance)?

## 5. Styling/visual checklist

- [ ] Desktop/Tablet/Mobile — chat window height/layout should adapt sensibly; check it doesn't get
  crushed to an unusable size on Mobile.
- [ ] Dark theme consistency with the rest of the page.

## 6. Widget-specific edge cases

- [ ] Leave Welcome title/subtitle blank — confirm the chat window's own sensible defaults show
  (not literally blank/empty text where a welcome message should be).
- [ ] Test with an ExecutionTemplateID set — confirm the first message is actually pre-hydrated with
  that template's input data as described.
