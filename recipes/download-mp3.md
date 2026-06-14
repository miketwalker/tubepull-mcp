# Recipe: download as MP3 / audio

Works on every supported platform — YouTube, TikTok, Vimeo, Dailymotion,
Twitter/X, SoundCloud, Bandcamp, Mixcloud — but
is the most common ask for audio-only platforms (SoundCloud, Bandcamp,
Mixcloud) and for music videos on YouTube.

## Trigger phrases

- "download this as MP3"
- "save this song"
- "extract the audio"
- "rip the audio"
- "get just the audio from this"
- "convert this to MP3"
- "save this Mixcloud set"
- "grab this SoundCloud track"
- "download this Bandcamp song"

## When to prefer `format: "audio"` over `format: "mp3"`

If the user just wants the audio and didn't name a format, send `audio`. It's
M4A with no transcoding — faster, smaller file, lossless to the source. Only
send `mp3` when the user explicitly says "MP3" (typically because they need
legacy device compatibility).

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
      "platform": "youtube",
      "title": "Rick Astley - Never Gonna Give You Up",
      "channel": "Rick Astley",
      "durationSec": 213,
      "expiresAt": "2026-05-11T09:46:00Z"
    }
  }
}
```

## Audio-only platforms (SoundCloud, Bandcamp, Mixcloud)

For these platforms you can omit `format` entirely — the server defaults to
`mp4`, then coerces to `m4a` because the platform has no video stream. The
response will show `format: "m4a"`. If you want MP3 instead, pass
`format: "mp3"` explicitly.

```json
{
  "jsonrpc": "2.0",
  "id": 43,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://soundcloud.com/artist-name/track-slug",
      "format": "mp3"
    }
  }
}
```

## Curl equivalent

```bash
curl -sS -X POST https://tubepull.com/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
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
