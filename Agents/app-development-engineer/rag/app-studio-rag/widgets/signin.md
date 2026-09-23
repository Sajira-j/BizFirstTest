# `signin` — Sign In / Sign Out

Sign-in link when signed out; a user menu with sign-out when signed in. Safe-default (all fields
optional). Source: `widget-handlers-signin-widget\src\SigninWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `signInUrl` | string | No | falls back to `@passport/app-urls`'s `LOGIN_APP_URL` | Where "Sign In" links to when signed out. |

## Example

```json
{}
```

## Gotchas

- Leaving `signInUrl` unset is the normal case — the widget already knows the real login app URL.
