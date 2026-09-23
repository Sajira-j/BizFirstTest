# `video` — Video

A single video player — URL, optional poster/caption, autoplay/loop/controls. **Modal-required**
(`videoUrl` has no sensible default). Source: `widget-handlers-video-widget\src\VideoWidgetConfig.ts`.

## Config

| Field (JSON key) | Type | Required | Default | Notes |
|---|---|---|---|---|
| `videoUrl` | string | **Yes** | — | Set via the Media Source picker, same pattern as `image`. |
| `posterUrl` | string | No | browser's own default (first frame once loaded, or blank until then) | Thumbnail shown before playback. |
| `caption` | string | No | — | Shown below the video if set. |
| `autoplay` | boolean | No | `false` | Never default-on — most browsers block audible autoplay anyway. |
| `loop` | boolean | No | `false` | |
| `controls` | boolean | No | `true` | A video with no controls and no autoplay is unplayable — don't set both `controls: false` and `autoplay: false`. |

## Example

```json
{ "videoUrl": "https://.../clip.mp4", "controls": true, "autoplay": false }
```

## Gotchas

- Same no-thumbnail-generation caveat as `image` — the original file is always served.
