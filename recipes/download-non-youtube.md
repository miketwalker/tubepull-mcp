# Recipe: download from non-YouTube platforms

TubePull supports seven platforms beyond YouTube. The MCP surface is identical
to YouTube — same `download_video` tool, same `url` argument. Just pass the
URL and TubePull detects the platform automatically.

## Supported platforms

| Platform              | Example URL                                                                  | Medium |
| --------------------- | ---------------------------------------------------------------------------- | ------ |
| TikTok                | `https://www.tiktok.com/@username/video/1234567890`                          | video  |
| Vimeo                 | `https://vimeo.com/123456789`                                                | video  |
| Dailymotion           | `https://www.dailymotion.com/video/x7tgad0`                                  | video  |
| Twitter / X           | `https://twitter.com/user/status/1234567890` (status must contain a video)   | video  |
| SoundCloud            | `https://soundcloud.com/artist/track-slug`                                   | audio  |
| Bandcamp              | `https://artist.bandcamp.com/track/song-slug`                                | audio  |
| Mixcloud              | `https://www.mixcloud.com/channel/show-slug/`                                | audio  |

## Trigger phrases per platform

The chatbot should pick `download_video` whenever the user says any download
verb plus a URL from one of these hosts. Examples:

- "save this TikTok"
- "grab this Vimeo"
- "download this Twitter video"
- "rip this SoundCloud track"
- "save this Bandcamp song"
- "download this Mixcloud set"

## Example: TikTok

```json
{
  "jsonrpc": "2.0",
  "id": 100,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://www.tiktok.com/@username/video/7234567890123456789"
    }
  }
}
```

Successful response includes `platform: "tiktok"` in `structuredContent`.

## Example: SoundCloud (audio-only platform)

You can omit `format` — the server's default of `mp4` is coerced to `m4a`
because SoundCloud has no video stream. To request MP3, pass it explicitly:

```json
{
  "jsonrpc": "2.0",
  "id": 102,
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

## Example: Mixcloud (audio-only, often long-form DJ mixes)

```json
{
  "jsonrpc": "2.0",
  "id": 103,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://www.mixcloud.com/channel-name/show-slug/"
    }
  }
}
```

Mixcloud sets are often 1–3 hours long. Free-tier users will hit
`paywall_duration` on anything over 2 hours — surface `upgradeUrl` from the
response.

## What `failure_reason` to expect

| `failure_reason`        | What to tell the user                                                      |
| ----------------------- | -------------------------------------------------------------------------- |
| `unsupported_platform`  | "TubePull doesn't support that site. Supported: YouTube, TikTok, Vimeo, Dailymotion, Twitter/X, SoundCloud, Bandcamp, Mixcloud." |
| `unsupported_url_shape` | "That looks like a profile / playlist / channel page. Send a single track or video URL." |
| `probe_error`           | Likely region-locked, private, or removed. Tell the user.                  |
| `fetch_error`           | Suggest retrying or trying a different quality.                            |

## Important: check `platform` in the response

`download_video` returns `platform` in `structuredContent`. Read it. The
server's URL detection is authoritative — for example, a `youtu.be` short
link still resolves to `platform: "youtube"`, and a `vm.tiktok.com` short
link resolves to `platform: "tiktok"`. Don't try to re-derive the platform
from the URL the user typed.
