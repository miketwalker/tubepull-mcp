# Recipe: download as MP4 (video)

Works on every video platform TubePull supports: YouTube, TikTok, Reddit,
Vimeo, Dailymotion, Rumble, Twitch, Twitter/X. Audio-only platforms
(SoundCloud, Bandcamp, Mixcloud) ignore `format: "mp4"` and return audio
instead — see `download-mp3.md` for those.

## Trigger phrases

- "download this video" / "save this video"
- "save this YouTube as MP4" / "save this TikTok" / "save this Reddit video"
- "save this clip offline" / "archive this video"
- "give me this in 1080p" / "download in 4K"
- "save this Twitch clip" / "download this Vimeo video"

## Choosing a quality

`quality` is optional. If the user named one (e.g. "in 720p"), pass it
through. Otherwise omit and TubePull picks the best available within their
tier. Reminder:

- `360p`, `480p`, `720p`, `1080p` — available on **all tiers**.
- `1440p`, `2160p` (4K) — **Unlimited only**. Free-tier calls returning a
  `paywall_quality` failure should surface the `upgradeUrl` from the
  response.

If you want to avoid the paywall round-trip on uncertain content, call
`get_video_info` first and check `qualities[]`.

## JSON-RPC call

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
      "format": "mp4",
      "quality": "1080p"
    }
  }
}
```

## Successful response (abridged)

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "content": [
      { "type": "text", "text": "Your MP4 (1080p) is ready: https://tubepull.com/dl/xyz789" }
    ],
    "structuredContent": {
      "downloadUrl": "https://tubepull.com/dl/xyz789",
      "filename": "Rick Astley - Never Gonna Give You Up [1080p].mp4",
      "format": "mp4",
      "quality": "1080p",
      "platform": "youtube",
      "title": "Rick Astley - Never Gonna Give You Up",
      "channel": "Rick Astley",
      "durationSec": 213,
      "expiresAt": "2026-05-11T09:46:00Z"
    }
  }
}
```

## Paywall failure (free user asking for 4K)

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "isError": true,
    "content": [
      { "type": "text", "text": "4K downloads require an Unlimited subscription. Upgrade at https://tubepull.com (only $3.99/mo)." }
    ],
    "structuredContent": {
      "failure_reason": "paywall_quality",
      "requestedQuality": "2160p",
      "upgradeUrl": "https://tubepull.com/?utm_source=mcp&utm_campaign=paywall_quality"
    }
  }
}
```

Surface `upgradeUrl` directly to the user — the link is pre-stamped with
attribution so the conversion is credited to your platform.

## Curl equivalent

```bash
curl -sS -X POST https://tubepull.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc":"2.0","id":7,"method":"tools/call",
    "params":{
      "name":"download_video",
      "arguments":{
        "url":"https://www.youtube.com/watch?v=dQw4w9WgXcQ",
        "format":"mp4",
        "quality":"1080p"
      }
    }
  }'
```
