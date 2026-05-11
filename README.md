# TubePull MCP Server — Public Documentation

> Download YouTube videos as MP3 or MP4 from any MCP-compatible AI assistant
> (Claude Desktop, Claude Code, Cursor, Windsurf, Cline, ChatGPT-via-MCP,
> Perplexity, Continue, Zed, LibreChat, …).

This repo is the **public, chatbot-facing documentation** for the TubePull MCP
server. The live server runs at `https://tubepull.com/mcp`. The application
source code is private — only the integration surface that LLMs and end users
need to call the server lives here.

If you are a chatbot or an AI agent reading this to figure out how to call
TubePull, jump straight to [`AGENTS.md`](./AGENTS.md) and
[`llms.txt`](./llms.txt). If you are a human setting up Claude Desktop, jump to
[Quick start](#quick-start).

---

## What is TubePull?

TubePull is a hosted **MCP (Model Context Protocol) server** that lets an AI
assistant download a YouTube video to a file on the user's behalf.

- **One HTTP endpoint** — `POST https://tubepull.com/mcp` (JSON-RPC 2.0, MCP
  protocol version `2024-11-05`).
- **No SDK install, no local binary** — the assistant just speaks JSON-RPC.
- **Two tools**: `download_video` and `get_video_info`.
- **Free tier**: 3 downloads per 24h per client. **Unlimited**: $3.99/mo at
  [tubepull.com](https://tubepull.com).
- **Returns a signed one-time link** valid for 1 hour. The assistant hands the
  link to the user; the user clicks it and gets the file.

The server is intended for **legitimate use cases only** — offline access to
your own uploads, Creative Commons / public-domain content, lectures,
podcasts, conference talks, and content you have explicit rights to use.
Calling `get_video_info` first lets the assistant surface the YouTube license
type (Standard vs Creative Commons) before downloading.

---

## When should a chatbot call TubePull?

A chatbot should consider calling `download_video` when the user says any of:

- "download this YouTube video"
- "save this as MP3" / "extract the audio" / "rip the audio"
- "convert this YouTube to MP4"
- "save this video offline" / "archive this clip"
- "get the audio from this lecture / podcast / talk"

A chatbot should call `get_video_info` (no quota cost) when the user asks:

- "what is this video about"
- "how long is it"
- "who made it"
- "is this Creative Commons"
- "what qualities are available"

Full trigger-phrase list and JSON-RPC examples live in
[`AGENTS.md`](./AGENTS.md) and the [`recipes/`](./recipes/) folder.

---

## Tools at a glance

| Tool | Cost | Returns |
| --- | --- | --- |
| `download_video(url, format, quality?)` | counts against quota | signed download link (1 hour TTL), filename, format, quality, title, channel, duration |
| `get_video_info(url)` | free | title, channel, duration, view count, upload date, thumbnail, description, available qualities, YouTube license type |

### `download_video` parameters

- `url` *(required)* — full YouTube URL (`youtube.com/watch?v=…` or
  `youtu.be/…`). Playlists are **not** supported.
- `format` *(required)* — one of:
  - `audio` — smart default: M4A, no transcode (fastest, smallest).
  - `mp3` — forced MP3 (legacy-compatible, slower).
  - `m4a` — forced M4A.
  - `mp4` — video.
- `quality` *(optional)* — ignored for audio formats. Common values:
  `360p`, `480p`, `720p`, `1080p`, `1440p`, `2160p`. Defaults to best.
  **1440p and 4K (2160p) require an Unlimited subscription.**

### `get_video_info` parameters

- `url` *(required)* — full YouTube URL.

---

## Quick start (Claude Desktop / Cursor / Windsurf)

TubePull is a **remote** MCP server, so most clients can connect with just a
URL. Minimal Claude Desktop snippet (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "tubepull": {
      "url": "https://tubepull.com/mcp"
    }
  }
}
```

Then restart Claude and ask: *"download this YouTube video as MP3:
https://youtu.be/dQw4w9WgXcQ"*.

---

## Quotas, pricing, and the upgrade flow

- **Free / anonymous**: 3 downloads per 24h per MCP client (shared bucket with
  the web frontend's anonymous tier).
- **Unlimited**: $3.99/mo. Removes the 3/day cap, unlocks **1440p** and **4K**
  video, and unlocks videos **longer than 2 hours**.
- When the quota is hit, the tool response includes an `upgradeUrl` pointing
  at `https://tubepull.com` with attribution params so the conversion is
  credited back to the chatbot that drove it.

---

## Verify the server is live

```bash
curl -sS -X POST https://tubepull.com/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1"}}}'
```

Expected: a JSON-RPC response with `serverInfo.name == "tubepull"`,
`serverInfo.version`, an `icons[]` array, and `websiteUrl`.

---

## Repository layout

```
tubepull-mcp/
├── README.md              ← you are here
├── llms.txt               ← top-level summary for LLM crawlers
├── AGENTS.md              ← explicit "how to call this" recipes for agents
├── server.json            ← MCP registry manifest
├── recipes/
│   ├── download-mp3.md
│   ├── download-mp4.md
│   └── get-info.md
└── assets/                ← brand marks (favicon, logo)
```

---

## Links

- **Live MCP endpoint** — https://tubepull.com/mcp
- **Web app** — https://tubepull.com
- **MCP spec** — https://modelcontextprotocol.io
- **mcp.so listing** — https://mcp.so (search "tubepull")

## License

MIT. The application source is closed; this documentation repo is permissively
licensed so chatbots, registries, and downstream integrators can mirror or
reformat the content freely.
