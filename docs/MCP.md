# Editing Lumae projects from an agent (MCP)

Lumae is a Model Context Protocol server. An MCP client such as Claude Code or
Claude Desktop can list your projects, read them, and edit them: cut clips, add
zooms, write captions, import media, undo. Every change an agent makes goes
through the same editing engine as your own clicks, so it shows up live in the
editor window, is one step in Edit ▸ Undo, and autosaves like anything else.

## Connect a client

The server is the helper inside the app bundle:

```
/Applications/Lumae.app/Contents/MacOS/lumae-mcp
```

Settings › MCP Server shows the exact path for your copy and prints the setup
below for each client with a Copy button. Lumae does not need to be running:
the helper launches it in the background when a client connects.

**Claude Code**

```bash
claude mcp add lumae -- /Applications/Lumae.app/Contents/MacOS/lumae-mcp
```

Then, in a Claude Code session: *"List my Lumae projects"*, *"Open the
onboarding demo and cut it at 12 seconds"*, *"Add a zoom on the click at 0:08
that follows the cursor"*.

**Claude Desktop**

Settings › Developer › Edit Config, then merge:

```json
{
  "mcpServers": {
    "lumae": { "command": "/Applications/Lumae.app/Contents/MacOS/lumae-mcp" }
  }
}
```

Restart Claude Desktop.

**Codex CLI**

Add to `~/.codex/config.toml`:

```toml
[mcp_servers.lumae]
command = "/Applications/Lumae.app/Contents/MacOS/lumae-mcp"
```

**Cursor**

Cursor Settings › MCP, or `~/.cursor/mcp.json`, with the same JSON as Claude
Desktop.

**Any other client** that speaks MCP over stdio: run the helper as the server
command with no arguments.

## What an agent can do

| Tool | What it does |
|---|---|
| `list_projects` | The open projects (frontmost first) and everything in the library, with ids. |
| `get_project` | One project: a summary in seconds (clips with their timeline and media times, zooms, captions, narration, assets) and the exact document the edit commands use. Works for projects that are not open. |
| `open_project` | Opens a project's window, by id or by the path of a `.lumae` package. |
| `edit_project` | Applies edit commands atomically as one undo step. Every edit the app can make is a command: rename, canvas, look (and one clip's own look with `setClipStyle`: background, padding, corners or shadow that override the video's for that clip), insert/move/delete/trim/split/replace clips, volume and mute, playback speed (`setClipSpeed`, 0.25…4, pitch kept), zooms (a zoom sent without a `transition` inherits the video's `style.motion`; zoom and click times in the summary are already on the timeline, through the clip's rate), narration, captions. |
| `preview_edit` | Validates commands and returns the result without changing anything. |
| `import_media` | Imports video files as new clips at the end of the timeline. |
| `split_at` | Cuts whatever plays at a timeline time. |
| `add_zoom_at` | Adds a zoom around a timeline time, optionally following the recorded cursor. |
| `auto_zoom` | Zooms where the recorded cursor clicked, like the toolbar's Automatic Zoom. |
| `render_frame` | Shows a frame as an image: the composed picture the export would show, or the raw recording. |
| `sample_frames` | Several frames in one labelled grid, to survey a project cheaply. |
| `export_project` | Renders the video to a file under `~/Movies` and returns its path. |
| `undo`, `redo` | Edit ▸ Undo and Redo for the project. |
| `describe_commands` | The command reference: every command with an example. |

Agents do not need a project id when only one editor window is open; the
frontmost one is the default. Editing a project that is not open opens it.

**Seeing the project.** `get_project` lists, for every clip, where and when the
recorded cursor clicked, so an agent can aim a zoom without looking. When it
does need to look, `render_frame` and `sample_frames` return JPEG images that
clients such as Claude Code display and reason over, and `preview_edit` with
`renderAt` shows what a change *would* look like before it is applied. A frame
is about 500 tokens at the default width; a six-frame grid costs about the same
as one frame.

**Exporting.** `export_project` writes the video through the same pipeline as
the Export sheet. The app is sandboxed, so files can only go under `~/Movies`
(default `~/Movies/Lumae Exports/<project name>.mp4`); ask the agent to move the
file from there if it belongs elsewhere. The call returns when the file is
complete, which can take a while for long projects.

Not available yet: recording, screenshots.

## How it works

The app hosts the server (the official MCP Swift SDK's `Server`) and listens
on a Unix socket inside its sandbox container. The helper relays the client's
stdio to that socket, launches the app when it is not running, and stages
files named by `import_media` into the container so the sandboxed app is
allowed to read them (the app removes the staged copies after importing). Only
your user account can reach the socket.

## Troubleshooting

- **"Lumae is not available"** (JSON-RPC error -32000): the helper could not
  reach the app within 20 seconds. Check that Lumae launches on its own, and
  that the helper you configured is the one inside the Lumae you run
  (Settings › MCP Server shows the path).
- **The helper's diagnostics** go to its stderr, which most clients show in
  their MCP logs: `lumae-mcp: Lumae is not running; launching it`,
  `connected to Lumae`, `could not reach Lumae: …`.
- **The app's side** logs to the unified log:
  `log show --predicate 'subsystem == "coderlife.Lumae" AND category == "mcp"' --last 1h --info`.
- **A Lumae somewhere other than /Applications:** `LUMAE_MCP_APP=/path/to/Lumae.app`
  tells the helper which copy to launch; `LUMAE_MCP_SOCKET` overrides the socket
  path.
- **Undo did more than one edit:** every request is its own undo step. If two
  edits ever merge into one, that is a bug — please report it.

## Privacy

Analytics record, in the shape PostHog's MCP Analytics reads, that a client
connected (its name and version, e.g. `claude-code`), that it listed the tools,
and which tool was called with what outcome, error category and duration, plus
the shape of its arguments (which commands, what numbers, which options, with
every piece of text such as names, captions and paths replaced by a
placeholder) and the one-sentence reason the agent gives for the call, which
every tool asks for in its `context` argument. Never the results or the project
itself. Nothing about your projects leaves the Mac through MCP
except what the connected client asks for and shows you.
