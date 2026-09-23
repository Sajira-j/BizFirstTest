# `audio` — Audio

A single audio player — URL, optional title/caption, autoplay/loop. **Modal-required** (`audioUrl`
has no sensible default). Source: `widget-handlers-audio-widget\src\AudioWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `audioUrl` | string | **Yes** | — | Set via the Media Source picker. |
| `title` | string | No | — | Audio's equivalent of Image's alt text / Video's poster — a label shown above the player identifying what's playing. |
| `caption` | string | No | — | Shown below the player if set. |
| `autoplay` | boolean | No | `false` | Never default-on. |
| `loop` | boolean | No | `false` | |

## Example

```json
{ "audioUrl": "https://.../track.mp3", "title": "Q3 town hall recording" }
```
