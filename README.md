# Lumae

Lumae turns a screen recording into a polished software demo on macOS. Record a
window, a display or an area; Lumae follows your cursor, zooms in on what you
clicked, and renders it over a background you choose. Nothing is re-encoded
until you export.

- **Zooms that follow the cursor.** Lumae records where you clicked and can place
  the zooms for you, or you can place them by hand on the timeline.
- **A look per project or per clip.** Background, padding, corner radius and
  shadow, with motion defaults you set once.
- **Captions and voiceover** on the timeline, mixed against the clip audio.
- **Exports** H.264 or HEVC, MP4 or MOV, at the size you need.

## Install

Download the latest `Lumae-<version>.dmg` from the
[Releases page](https://github.com/matlegault/lumae-releases/releases/latest),
open it, and drag Lumae into your Applications folder. Builds are signed and
notarized. Requires macOS 26 or later.

Installed copies update themselves through Sparkle; the `.zip` and `.delta`
files attached to each release are what the updater downloads.

## Edit your demos from a coding agent

Lumae is an [MCP](https://modelcontextprotocol.io) server, so an agent such as
Claude Code, Claude Desktop, Cursor or Codex can work on your recordings
directly:

> *"Open the onboarding demo, cut the dead air at 12 seconds, and add a zoom on
> the click at 0:08 that follows the cursor."*

Every change an agent makes goes through the same editing engine as your own
clicks, so it appears live in the editor window, is one step in Edit ▸ Undo, and
autosaves like anything else. The agent can also render frames to look at what
it is doing before it commits.

Connect Claude Code:

```bash
claude mcp add lumae -- /Applications/Lumae.app/Contents/MacOS/lumae-mcp
```

For Claude Desktop, Cursor, Codex and any other MCP client, see
**[docs/MCP.md](docs/MCP.md)** — it also lists every tool an agent can call.
Settings ▸ MCP Server inside the app prints the configuration for each client
with a Copy button.

Nothing about your projects leaves your Mac through MCP except what the client
you connected asks for and shows you.

## Support

Questions and bug reports: [Issues](https://github.com/matlegault/lumae-releases/issues).
