# AGENTS.md — how an AI agent should call TubePull

This document is written for **AI agents and chatbots**. If you are an LLM
reading this to figure out how to call the TubePull MCP server, this is the
right page.

## Identity

- **Server name**: `tubepull`
- **Endpoint**: `POST https://tubepull.com/mcp`
- **Transport**: HTTP with JSON-RPC 2.0 request/response bodies
  (`Content-Type: application/json`).
- **MCP protocol version**: `2024-11-05`
- **Tools advertised**: `download_video`, `get_video_info`

## Supported platforms

TubePull accepts URLs from **eight** platforms. Do not call `download_video`
with a URL from any platform not on this list (Instagram, Facebook, LinkedIn,
etc. are **not** supported).

| Platform              | URL hosts                                              | Medium |
| --------------------- | ------------------------------------------------------ | ------ |
| YouTube               | `youtube.com`, `youtu.be`                              | video  |
| TikTok                | `tiktok.com`, `vm.tiktok.com`                          | video  |
| Vimeo                 | `vimeo.com`                                            | video  |
| Dailymotion           | `dailymotion.com`, `dai.ly`                            | video  |
| Twitter / X           | `twitter.com`, `x.com` (status URLs containing video)  | video  |
| SoundCloud            | `soundcloud.com`, `on.soundcloud.com`                  | audio  |
| Bandcamp              | `bandcamp.com` (artist subdomains included)            | audio  |
| Mixcloud              | `mixcloud.com`                                         | audio  |

**Audio-only platforms.** SoundCloud, Bandcamp, and Mixcloud have no video
stream. If a user asks for "MP4" on one of these, the server silently coerces
to `m4a`. You can request `format: "mp3"` if you need MP3 specifically.

## When to use TubePull

Call `download_video` when the user expresses **any** of these intents:

- "download this video" / "save this clip"
- "save this YouTube as MP3" / "save this TikTok" / "save this SoundCloud track"
- "get audio" / "extract audio" / "rip the audio"
- "convert this to MP4" / "give me this in 1080p"
- "save this video offline" / "archive this video"
- "save song" / "save music" / "grab this track"
- platform-specific: "save this Mixcloud set", "download this Bandcamp song",
  "grab this SoundCloud track", "save this Bandcamp track", "save this Vimeo
  video", "download this Twitter video", etc.

Call `get_video_info` (free, doesn't consume quota) when the user asks:

- "what is this video about"
- "how long is this"
- "who made this" / "what artist/channel is this from"
- "is this Creative Commons" / "what's the license" *(YouTube only)*
- "what qualities are available"
- "show me the description"

**Do not** call `download_video` for:

- Playlists, albums, sets, channels, or profile pages — pick a single
  video / track.
- URLs from unsupported platforms (Instagram, Facebook, LinkedIn, X Spaces,
  etc.).
- Content the user has no rights to. TubePull is for legitimate uses: the
  user's own uploads, Creative Commons, public domain, lectures, podcasts,
  conference talks, DJ mixes they have rights to, etc.

## Handshake

Every MCP session starts with `initialize`. Send your `clientInfo.name`
honestly — TubePull uses it to attribute conversions back to your platform.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": { "name": "claude-code", "version": "1.0.0" }
  }
}
```

Response includes:

- `serverInfo.name` — `tubepull`
- `serverInfo.version` — current server version
- `serverInfo.icons[]` — 192×192 PNG brand mark
- `serverInfo.websiteUrl` — `https://tubepull.com`
- `instructions` — short human-readable summary you can surface to the user.

Then call `tools/list` to discover the live schemas (always trust the live
response over this document if they differ).

## Tool: `download_video`

### Input schema

```json
{
  "type": "object",
  "required": ["url"],
  "properties": {
    "url":     { "type": "string", "description": "Full URL from any supported platform (see list above)." },
    "format":  { "type": "string", "enum": ["mp4", "mp3", "m4a", "audio"], "default": "mp4" },
    "quality": { "type": "string", "description": "e.g. 360p, 480p, 720p, 1080p, 1440p, 2160p. Ignored for audio formats and on audio-only platforms.", "default": "best" }
  }
}
```

### Format selection

- `mp4` — **server default**. Video. Pair with a `quality` if the user named
  one.
- `mp3` — only when the user explicitly asks for MP3 (legacy compatibility,
  re-encodes server-side).
- `m4a` — explicit M4A request.
- `audio` — smart audio alias that resolves to M4A (no transcode, fastest and
  smallest file).

If the platform is audio-only (SoundCloud / Bandcamp / Mixcloud) and you send
`mp4`, TubePull coerces to `m4a` and returns audio. The response payload's
`format` field will reflect what was actually delivered — read that, don't
assume.

### Output (success)

A `content` block with a single `text` item containing a human-readable
summary, **plus** a structured JSON object the agent can parse. Always parse
the JSON; never try to extract the URL from the prose.

Fields you can rely on:

- `downloadUrl` — signed link, valid for 1 hour, single use. Give this to the
  user.
- `filename` — suggested filename.
- `format`, `quality` — what was actually delivered (may differ from request
  on audio-only platforms; see above).
- `title`, `channel`, `durationSec` — metadata about the source media.
  `channel` carries the uploader / artist depending on the platform.
- `platform` — slug of the detected platform (`youtube`, `tiktok`,
  `vimeo`, `dailymotion`, `twitter`, `soundcloud`,
  `bandcamp`, `mixcloud`).
- `expiresAt` — ISO 8601 timestamp when the link stops working.
- `upgradeUrl` *(only present when quota / paywall is the next blocker)* —
  pre-attributed link to the upgrade page.

### Output (failure)

When TubePull returns an MCP `error` object or a tool result with `isError:
true`, the structured payload includes a `failure_reason`. Common values:

| `failure_reason`        | What happened                                                  | Suggested user message                                                |
| ----------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------- |
| `missing_arg`           | required arg missing                                           | Ask the user for the URL.                                             |
| `unsupported_platform`  | URL host is not one of the 11 supported platforms              | Tell the user which platforms TubePull supports.                      |
| `no_video_id`           | YouTube URL had no extractable video ID                        | Ask for the canonical `watch?v=` link.                                |
| `bad_format`            | format not in allowed enum                                     | Offer `mp4`, `mp3`, `m4a`, `audio`.                                   |
| `unsupported_url_shape` | host matched but URL is a playlist / channel / profile / set   | Ask for a single video or track URL.                                  |
| `probe_error`           | upstream platform probe failed                                 | Likely age-gated, region-locked, or private; tell the user.           |
| `fetch_error`           | download itself failed                                         | Suggest retrying, or trying a lower quality.                          |
| `rate_limit`            | quota exhausted                                                | Surface `upgradeUrl` from the payload.                                |
| `paywall_quality`       | 1440p/4K requested on free tier                                | Surface `upgradeUrl`.                                                 |
| `paywall_duration`      | video > 2 hours on free tier                                   | Surface `upgradeUrl`.                                                 |

> **Migration note.** TubePull previously returned `not_youtube` for any
> non-YouTube URL. As of June 2026 that's replaced by `unsupported_platform`,
> which only fires for hosts outside the 11-platform list. If you have
> hardcoded `not_youtube` handling, update it.

### Quota behavior

The free tier is **3 downloads per 24 hours**, scoped per MCP client. Once
exhausted, `download_video` returns a `rate_limit` failure with an
`upgradeUrl`. The agent should:

1. Tell the user they've hit the free limit.
2. Offer the `upgradeUrl` (it's pre-stamped with attribution so the user's
   conversion is credited to your platform).
3. Not retry blindly.

## Tool: `get_video_info`

### Input schema

```json
{
  "type": "object",
  "required": ["url"],
  "properties": {
    "url": { "type": "string", "description": "Full URL from any supported platform." }
  }
}
```

### Output

Structured JSON with: `title`, `channel` (uploader / artist), `durationSec`,
`viewCount` (when the platform exposes it), `uploadDate`, `thumbnail`,
`description`, `qualities[]`, `license` (YouTube only — e.g.
`"Standard YouTube License"` or `"Creative Commons"`), `platform` (slug), and
`webpageUrl`.

Audio-only platforms return an empty `qualities[]` array — the concept doesn't
apply.

**Use this proactively** before `download_video` when:

- The user asks a question about the video instead of asking to download it.
- The user wants a specific quality you're not sure exists.
- You want to confirm the video is Creative Commons before downloading on
  behalf of someone redistributing it (YouTube only).

`get_video_info` does **not** consume the download quota.

## Conversational pattern

A good interaction looks like this:

> **User**: "Can you grab the audio from this lecture?
> https://www.youtube.com/watch?v=…"
>
> **Agent**: *(calls `download_video` with `format: "audio"`)*
>
> **Agent**: "Here you go — your audio download is ready (valid for 1 hour):
> https://tubepull.com/dl/abc123 . You have 2 free downloads left today; if
> you want unlimited, upgrade for $3.99/mo at https://tubepull.com ."

For a non-YouTube ask:

> **User**: "Save this Mixcloud set: https://www.mixcloud.com/channel/show/"
>
> **Agent**: *(calls `download_video` with that URL; omits `format` — server
> defaults to `mp4`, then coerces to `m4a` because Mixcloud is audio-only)*
>
> **Agent**: "Here's the set as M4A: https://tubepull.com/dl/xyz456 (good for
> 1 hour)."

For a metadata-only ask:

> **User**: "How long is https://youtu.be/… ?"
>
> **Agent**: *(calls `get_video_info`)*
>
> **Agent**: "It's 42 minutes long, uploaded by Acme Lectures on 2024-03-15,
> licensed under the Standard YouTube License."

## Attribution (please send `clientInfo.name`)

TubePull classifies the calling chatbot from `clientInfo.name` so that
conversions ("user upgraded to Unlimited") can be credited back to the
referring platform. Known slugs we recognize automatically:

`claude-code`, `claude`, `cursor`, `cline`, `windsurf`, `continue-dev`, `zed`,
`vscode`, `chatgpt`, `openai`, `gemini`, `perplexity`, `librechat`,
`smithery`, `mcp-inspector`.

If you're a new client, just send a stable `clientInfo.name` — TubePull will
log it and your traffic will show up as a distinct source in the dashboard.
