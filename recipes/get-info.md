# Recipe: get video / audio metadata (no download)

Works on every platform TubePull supports.

## Trigger phrases

- "what is this video about"
- "how long is this"
- "who made this" / "what artist/channel is this"
- "is this Creative Commons" / "what's the license" *(YouTube only)*
- "what qualities are available"
- "show me the description"

## Why this exists

`get_video_info` is **free** — it does **not** count against the 3/day
download quota. Use it any time the user asks a question about a video or
track rather than asking you to download it, or when you need to decide on a
`quality` before calling `download_video`.

## JSON-RPC call

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "method": "tools/call",
  "params": {
    "name": "get_video_info",
    "arguments": {
      "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
    }
  }
}
```

## Response (abridged)

```json
{
  "jsonrpc": "2.0",
  "id": 11,
  "result": {
    "content": [
      { "type": "text", "text": "Rick Astley - Never Gonna Give You Up · Rick Astley · 3:33 · Standard YouTube License" }
    ],
    "structuredContent": {
      "title": "Rick Astley - Never Gonna Give You Up",
      "channel": "Rick Astley",
      "durationSec": 213,
      "viewCount": 1500000000,
      "uploadDate": "2009-10-25",
      "thumbnail": "https://i.ytimg.com/vi/dQw4w9WgXcQ/maxresdefault.jpg",
      "description": "The official music video for…",
      "qualities": ["144p", "240p", "360p", "480p", "720p", "1080p"],
      "license": "Standard YouTube License",
      "platform": "youtube",
      "webpageUrl": "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
    }
  }
}
```

## Per-platform field availability

Not every platform exposes every field. Treat fields as best-effort:

| Field | YouTube | TikTok | Vimeo | Dailymotion | Twitter | SoundCloud | Bandcamp | Mixcloud |
| ------------- | ------- | ------ | ----- | ----------- | ------- | ---------- | -------- | -------- |
| `title` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `channel` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `durationSec` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `viewCount` | ✓ | ✓ | ✓ | ✓ | ~ | ✓ | ~ | ✓ |
| `uploadDate` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `qualities[]` | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ (audio) | ✗ (audio) | ✗ (audio) |
| `license` | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

`~` = sometimes available, sometimes not.

## Curl equivalent

```bash
curl -sS -X POST https://tubepull.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{
    "jsonrpc":"2.0","id":11,"method":"tools/call",
    "params":{
      "name":"get_video_info",
      "arguments":{
        "url":"https://www.youtube.com/watch?v=dQw4w9WgXcQ"
      }
    }
  }'
```

## Tips for chaining with `download_video`

1. Call `get_video_info` first.
2. If `license` is `"Creative Commons"` (YouTube only), you can comfortably
   download on behalf of users who plan to redistribute.
3. If the user wants "the best quality available", read `qualities[]` and
   pick the highest one allowed on their tier (1080p free, 2160p Unlimited).
4. If `durationSec > 7200` (2 hours) and you don't know whether the user is
   Unlimited, warn them — free tier will hit `paywall_duration`.
