# `signin`

Source: `packages/widget-handlers-signin-widget`, `WidgetRecord.ts`'s own doc comment (dated
2026-08-28, "app-studio-widget-library-expansion"), scanned 2026-08-30.

Thin wrapper consuming `@bizfirst/common-auth-react`'s sign-in UI as an embeddable App Studio widget —
lets a page host a real login control inline rather than only via the full-page `SsoAuthGate`
redirect flow (see `architecture.md` §5 for the underlying SSO/handoff-code mechanism this widget
almost certainly drives).

## Config — OPEN QUESTION

Field shapes not independently re-verified this pass. Re-grep the handler package's own config type
(likely a post-login redirect target, and possibly styling/branding overrides) before generating
config for this widget type.
