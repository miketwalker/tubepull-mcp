# Recipe: download a YouTube video as MP3

## Trigger phrases

- "download this YouTube as MP3"
- "save this song"
- "extract the audio"
- "rip the audio"
- "get just the audio from this"
- "convert this to MP3"

## When to prefer `format: "audio"` over `format: "mp3"`

If the user just wants the audio and didn't name a format, send `audio`. It's
M4A with no transcoding — faster, smaller file, lossless to the YouTube
source. Only send `mp3` when the user explicitly says "MP3" (typically
because they need legacy device compatibility).

## JSON-RPC call

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
      "format": "mp3"
    }
  }
}
```

## Successful response (abridged)

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Your MP3 is ready: https://tubepull.com/dl/abc123 (valid 1 hour). 2 free downloads left today."
      }
    ],
    "structuredContent": {
      "downloadUrl": "https://tubepull.com/dl/abc123",
      "filename": "Rick Astley - Never Gonna Give You Up.mp3",
      "format": "mp3",
      "title": "Rick Astley - Never Gonna Give You Up",
      "channel": "Rick Astley",
      "durationSec": 213,
      "expiresAt": "2026-05-11T09:46:00Z"
    }
  }
}
```

## Curl equivalent

```bash
curl -sS -X POST https://tubepull.com/mcp \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0","id":42,"method":"tools/call",
    "params":{
      "name":"download_video",
      "arguments":{
        "url":"https://www.youtube.com/watch?v=dQw4w9WgXcQ",
        "format":"mp3"
      }
    }
  }'
```

## Suggested chat response

Hand the user the `downloadUrl` plus the remaining-quota / upgrade context:

> "Here's your MP3: https://tubepull.com/dl/abc123 (good for 1 hour). You
> have 2 free downloads left today — unlimited is $3.99/mo at
> https://tubepull.com if you want to lift the cap."
