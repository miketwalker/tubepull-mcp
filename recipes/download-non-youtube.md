# Recipe: download from non-YouTube platforms

TubePull supports ten platforms beyond YouTube. The MCP surface is identical
to YouTube — same `download_video` tool, same `url` argument. Just pass the
URL and TubePull detects the platform automatically.

## Supported platforms

| Platform              | Example URL                                                                  | Medium |
| --------------------- | ---------------------------------------------------------------------------- | ------ |
| TikTok                | `https://www.tiktok.com/@username/video/1234567890`                          | video  |
| Reddit                | `https://www.reddit.com/r/sub/comments/abc/title/`                           | video  |
| Vimeo                 | `https://vimeo.com/123456789`                                                | video  |
| Dailymotion           | `https://www.dailymotion.com/video/x7tgad0`                                  | video  |
| Rumble                | `https://rumble.com/v1abc-title.html`                                        | video  |
| Twitch (clip)         | `https://clips.twitch.tv/AwkwardHelplessSalamanderSwiftRage`                 | video  |
| Twitch (VOD)          | `https://www.twitch.tv/videos/1234567890`                                    | video  |
| Twitter / X           | `https://twitter.com/user/status/1234567890` (status must contain a video)   | video  |
| SoundCloud            | `https://soundcloud.com/artist/track-slug`                                   | audio  |
| Bandcamp              | `https://artist.bandcamp.com/track/song-slug`                                | audio  |
| Mixcloud              | `https://www.mixcloud.com/channel/show-slug/`                                | audio  |

## Trigger phrases per platform

The chatbot should pick `download_video` whenever the user says any download
verb plus a URL from one of these hosts. Examples:

- "save this TikTok"
- "download this Reddit video"
- "grab this Vimeo"
- "save this Twitch clip"
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

## Example: Reddit

```json
{
  "jsonrpc": "2.0",
  "id": 101,
  "method": "tools/call",
  "params": {
    "name": "download_video",
    "arguments": {
      "url": "https://www.reddit.com/r/funny/comments/abc123/some_title/"
    }
  }
}
```

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
| `unsupported_platform`  | "TubePull doesn't support that site. Supported: YouTube, TikTok, Reddit, Vimeo, Dailymotion, Rumble, Twitch, Twitter/X, SoundCloud, Bandcamp, Mixcloud." |
| `unsupported_url_shape` | "That looks like a profile / playlist / channel page. Send a single track or video URL." |
| `probe_error`           | Likely region-locked, private, or removed. Tell the user.                  |
| `fetch_error`           | Suggest retrying or trying a different quality.                            |

## Important: check `platform` in the response

`download_video` returns `platform` in `structuredContent`. Read it. The
server's URL detection is authoritative — for example, a `youtu.be` short
link still resolves to `platform: "youtube"`, and a `vm.tiktok.com` short
link resolves to `platform: "tiktok"`. Don't try to re-derive the platform
from the URL the user typed.
