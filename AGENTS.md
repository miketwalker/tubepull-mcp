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

## When to use TubePull

Call `download_video` when the user expresses **any** of these intents:

- "download this YouTube video"
- "save this YouTube as MP3"
- "get YouTube audio" / "extract audio from YouTube" / "rip the audio"
- "convert YouTube to MP4"
- "save this video offline" / "archive this video" / "download this clip"
- "save song" / "save music" (from a YouTube URL)

Call `get_video_info` (free, doesn't consume quota) when the user asks:

- "what is this video about"
- "how long is this video"
- "who made this video"
- "is this Creative Commons" / "what's the license"
- "what qualities are available"
- "show me the description"

**Do not** call `download_video` for:

- Playlist URLs (not supported — pick an individual video).
- Non-YouTube URLs (only `youtube.com/watch?v=…` and `youtu.be/…`).
- Content the user has no rights to. TubePull is for legitimate uses: the
  user's own uploads, Creative Commons, public domain, lectures, podcasts,
  conference talks, etc.

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
  "required": ["url", "format"],
  "properties": {
    "url":     { "type": "string", "description": "Full YouTube URL." },
    "format":  { "type": "string", "enum": ["audio", "mp3", "m4a", "mp4"] },
    "quality": { "type": "string", "description": "e.g. 720p, 1080p, 1440p, 2160p. Ignored for audio formats." }
  }
}
```

### Format selection

- `audio` — **default for audio intents**. Smart M4A, no transcoding, fastest
  and smallest file. Use this unless the user specifically says "MP3".
- `mp3` — only when the user explicitly asks for MP3 (legacy compatibility,
  re-encodes server-side, slightly slower).
- `m4a` — explicit M4A request.
- `mp4` — video. Pair with a `quality` if the user named one.

### Output (success)

A `content` block with a single `text` item containing a human-readable
summary, **plus** a structured JSON object the agent can parse. Always parse
the JSON; never try to extract the URL from the prose.

Fields you can rely on:

- `downloadUrl` — signed link, valid for 1 hour, single use. Give this to the
  user.
- `filename` — suggested filename.
- `format`, `quality` — what was actually delivered.
- `title`, `channel`, `durationSec` — metadata about the source video.
- `expiresAt` — ISO 8601 timestamp when the link stops working.
- `upgradeUrl` *(only present when quota / paywall is the next blocker)* —
  pre-attributed link to the upgrade page.

### Output (failure)

When TubePull returns an MCP `error` object or a tool result with `isError:
true`, the structured payload includes a `failure_reason`. Common values:

| `failure_reason` | What happened | Suggested user message |
| --- | --- | --- |
| `missing_arg` | required arg missing | Ask the user for the URL or format. |
| `not_youtube` | not a YouTube URL | Tell the user only YouTube URLs are supported. |
| `no_video_id` | URL had no video id | Ask for the canonical `watch?v=` link. |
| `bad_format` | format not in allowed enum | Offer `mp3`, `m4a`, `mp4`, `audio`. |
| `probe_error` | YouTube probe failed | Likely age-gated or region-locked; tell the user. |
| `fetch_error` | download itself failed | Suggest retrying, or trying a lower quality. |
| `rate_limit` | quota exhausted | Surface `upgradeUrl` from the payload. |
| `paywall_quality` | 1440p/4K requested on free tier | Surface `upgradeUrl`. |
| `paywall_duration` | video > 2 hours on free tier | Surface `upgradeUrl`. |

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
    "url": { "type": "string", "description": "Full YouTube URL." }
  }
}
```

### Output

Structured JSON with: `title`, `channel`, `durationSec`, `viewCount`,
`uploadDate`, `thumbnail`, `description`, `qualities[]`, `license` (e.g.
`"Standard YouTube License"` or `"Creative Commons"`), and `webpageUrl`.

**Use this proactively** before `download_video` when:

- The user asks a question about the video instead of asking to download it.
- The user wants a specific quality you're not sure exists.
- You want to confirm the video is Creative Commons before downloading on
  behalf of someone redistributing it.

`get_video_info` does **not** consume the download quota.

## Conversational pattern

A good interaction looks like this:

> **User**: "Can you grab the audio from this lecture?
> https://www.youtube.com/watch?v=…"
>
> **Agent**: *(calls `download_video` with `format: "audio"`)*
>
> **Agent**: "Here you go — your MP3 download is ready (valid for 1 hour):
> https://tubepull.com/dl/abc123 . You have 2 free downloads left today; if
> you want unlimited, upgrade for $3.99/mo at https://tubepull.com ."

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
