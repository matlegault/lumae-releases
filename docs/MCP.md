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
| `list_projects` | The open projects (frontmost first) and everything in the library, open ones included and flagged `isOpen`, with ids. Paths are reported through the sandbox symlink, so they read as `~/Movies/Lumae/…` rather than as a container, and a project's `path` follows its name — `renameProject` renames the `.lumae` package on disk too — so hold on to the id, not the path, across an edit. |
| `get_project` | One project: a summary in seconds (clips with their timeline and media times, zooms, captions, narration, music, camera takes with where each bubble lands on the canvas, assets) and the exact document the edit commands use. Works for projects that are not open. |
| `open_project` | Opens a project's window, by id or by the path of a `.lumae` package. |
| `edit_project` | Applies edit commands atomically as one undo step. Every edit the app can make is a command: rename, canvas, look (and one clip's own look with `setClipStyle`: background, padding, corners or shadow that override the video's for that clip), insert/move/delete/trim/split/replace clips, volume and mute, playback speed (`setClipSpeed`, 0.25…4, pitch kept), zooms (a zoom sent without a `transition` inherits the video's `style.motion`; zoom and click times in the summary are already on the timeline, through the clip's rate), narration (segments never overlap; they move, trim with `trimVoiceoverSegment`, split with `splitVoiceoverSegment` and fade with `setVoiceoverSegmentFades`, like music), music (`addMusicSegment` and friends: pieces on one track that never overlap, each with its own gain and fades; a piece may run past the video's end, where it is cut; `updateMusicMix` sets the track's level and ducking under narration, `setMusicMuted` silences it), the camera (`addCameraSegment` and friends: takes on one lane that never overlap and never run past the video's end, drawn as a bubble over the finished picture; `setCameraStyle` is the bubble every take follows — shape, size, which corner and how far in, stroke, shadow, mirroring — and `setCameraSegmentStyle` is one take's departures from it, an empty object following the video again), captions. |
| `preview_edit` | Validates commands and returns the result without changing anything. |
| | `edit_project` takes `summary: "brief"`, which reports the name, the duration and counts instead of the whole project — worth using during a run of edits, since the full summary repeats every zoom and caption back. |
| `import_media` | Video files become new clips at the end of the timeline. Audio files (mp3, m4a, wav, aiff) land on `lane`: `music`, after whatever is there and without fades (`setMusicSegmentFades` adds them; music needs a clip to sit under, being cut where the video ends), or `narration`, which is where a spoken track belongs and carries no fades because a line that fades in has lost its first word. Images (png, jpeg, heic, tiff, webp) become the video's background. |
| `split_at` | Cuts whatever plays at a timeline time, or, given `kind` and `id`, that clip, camera take, narration segment, piece of music or caption (zooms are not split). |
| `remove_range` | Cuts a span of timeline time out and closes the gap: clips inside it go, a clip straddling an edge is trimmed, one containing the whole span is split. Captions, narration, music and camera takes lose what they laid over the span and everything after is pulled back, so the edit stays in sync; zooms travel with their clip. `ranges` cuts several at once, last first, so the times stay right. |
| `add_zoom_at` | Adds a zoom around a timeline time, optionally following the recorded cursor. |
| `move_item` | Moves a zoom, a camera take, a narration segment, a piece of music or a caption to start at a timeline time, kept where its lane allows (between its neighbours, inside its clip for a zoom, before the video's end), and reports where it landed. Clips reorder with `moveClip` instead. |
| `trim_item` | Moves one edge of any item to a timeline time, kept inside its room, its media and its minimum length; the start edge of a clip, a narration segment or a piece of music shifts what plays along with it. |
| `auto_zoom` | Zooms where the recorded cursor clicked, like the toolbar's Automatic Zoom. |
| `list_capture_targets` | The displays and on-screen windows that can be recorded, windows front to back, each with its id and its frame in points, plus `selfCaptureAllowed` — Lumae's own windows are left out of the list while it is off. |
| `start_recording` | Records a display, a window, or a rectangle of a display, into a new project or an existing one. Returns once capture is running, and reports what the options settled on. The microphone stays off unless asked for, unlike every other setting, which follows the app's. |
| `stop_recording` | Stops the recording and returns once its clip is in the project, with the clip's id and the resulting summary. `trimToContent` cuts the take down to what happened. |
| `recording_state` | What the recorder is doing — idle, counting down, recording, finalizing — with seconds captured so far. Answered while a gesture is still running, so it can be polled throughout a take. |
| `perform_gesture` | Moves, clicks, drags, scrolls or types with a hand's pacing, while a recording runs. For the moments that are on camera. A `mark` step names a beat in the same call, so labelling costs no recording. Up to 64 steps, each `pause` up to 10 s and each `type` up to 500 characters. |
| `mark_moment` | Names what is happening in the running recording and timestamps it, for the edit afterwards. |
| `transcribe` | Every spoken word with its timeline timing, on-device. Optionally writes the captions. |
| `create_project` | An empty project in the library, opened. `start_recording` makes one on its own when no project is named. |
| `render_frame` | Shows a frame as an image: the composed picture the export would show, or the raw recording. |
| `sample_frames` | Several frames in one labelled grid, to survey a project cheaply. |
| `export_project` | Renders the video to a file under `~/Movies` and returns its path. H.264, HEVC, ProRes or GIF. |
| `undo`, `redo` | Edit ▸ Undo and Redo for the project. |
| `describe_commands` | The command reference: every command with an example. |
| `get_more_tools` | Reports a gap: an agent calls it when nothing here does what it was asked, describing the capability it wanted. Nothing in Lumae changes; the description is recorded so the tools can grow towards what agents actually need. |

Agents do not need a project id when only one editor window is open; the
frontmost one is the default. Editing a project that is not open opens it.

**Seeing the project.** `get_project` lists, for every clip, where and when the
recorded cursor clicked, so an agent can aim a zoom without looking. When it
does need to look, `render_frame` and `sample_frames` return JPEG images that
clients such as Claude Code display and reason over, and `preview_edit` with
`renderAt` shows what a change *would* look like before it is applied. Marks an edit has
cut away are not lost with it: `summary.cutMarks` keeps each one with where its
moment now falls, since a mark sits in the lead-in that gets removed first. A frame
is about 500 tokens at the default width; a six-frame grid costs about the same
as one frame.

**Exporting.** `export_project` writes the video through the same pipeline as
the Export sheet, but not to the same places. An agent's write has no save panel
behind it, so the only sandbox grant available is Lumae's `~/Movies` access, and
a destination outside it is refused with `destination_outside_movies` (default
`~/Movies/Lumae Exports/<project name>.mp4`). Ask the agent to move the file
afterwards if it belongs elsewhere. The Export sheet is not limited this way:
picking a file in the panel is itself the grant, so a person can export
anywhere, and the panel reopens wherever they last did. The call returns when
the file is complete, which can take a while for long projects.

`codec` picks the format: `h264` (plays everywhere), `hevc` (smaller),
`proRes422LT` / `proRes422` / `proRes422HQ` / `proRes4444` for handing to an
editor (always `.mov`, and large), or `gif` for something silent that loops in
a chat or a README. `quality` (`good` / `high` / `best`) applies to H.264 and
HEVC only. A GIF takes `gifFrameRate` and `gifColors`, and wants a small
`longEdge`, 720 or less, since GIF has no inter-frame compression to lean on.
`longEdge` is accepted between 240 and 7680 pixels, and a destination inside a
`.lumae` project package is refused (`destination_inside_package`) even under
`~/Movies`. `sample_frames` and `preview_edit`'s `renderAt` render at most 16
frames per call.

Not available yet: screenshots.

**Recording — Lumae films; the agent decides.** Nothing in the server chooses what to
demonstrate or where to click. This is worth saying out loud, because the tools alone read as
though Lumae might do the whole thing. The loop is: `list_capture_targets` → `start_recording`
→ *the app is worked while the recording runs* → `stop_recording` → edit → export.

Working the app can happen two ways. The client's own input is faster and needs no extra
permission — right for setup and for anything not being filmed. `perform_gesture` hands the
movement to Lumae, which performs it with a hand's pacing and only while a recording is
running — right for the moments on camera. The difference is timing that cannot be added
afterwards: `CursorTrack` fills in travel across a jump when it draws the pointer, but never
the pause before a click, because the video already shows the app reacting.

`list_capture_targets` reports what there is; `start_recording` points at one
of them and returns once pixels are actually being written, so no error means the capture is
under way. It records into a project you name or one it creates, and `stop_recording` returns
when the clip has landed, with the summary — so an agent can record and then edit without a
second lookup. Lumae records the cursor path and clicks alongside the video, which is what
`auto_zoom` and the drawn cursor use afterwards.

Front the window being driven before acting on it. Opening or creating a project brings a window
forward, which may land over the one the agent is working. Capture is unaffected — a window
filter captures its window whatever sits over it — but clicks are not, and a click on the wrong
window does nothing visible and reports nothing, which is a confusing way to lose several
minutes. `start_recording` raises the window it was pointed at once the project's own window has
opened, which covers the ordinary case; it cannot cover a window that something else raises
later, and activating another app is cooperative on macOS and may be declined. So check rather
than assume: `list_capture_targets` lists windows front to back with their frames, and a click
anywhere in a window brings it forward.

Whichever route, move in short eased steps rather than jumping the pointer about, and let a
moment pass before each click; a pointer that teleports had no path for Lumae to draw. Batching
steps into one `perform_gesture` call matters more than it looks: the recording runs in real
time, so every round trip an agent takes is dead video for someone to cut out later.

`perform_gesture` needs macOS's **Accessibility** permission as well, and refuses outside a
recording. That second limit is deliberate: it keeps the tool a way of making a filmed moment
look deliberate rather than a way to operate someone's Mac, and inside a recording the
countdown has already run and the HUD is on screen, so nothing moves the pointer without the
person at the machine knowing a capture is under way. It holds for the whole gesture, not only
its first step: `stop_recording` lands while one is running, and ends it where it stands rather
than leaving it driving the Mac over whatever comes next. The outcome then says how many steps
were left, which is worth reading before planning cuts from beats that may not have happened.

Everything a gesture takes is bounded by what a take can afford — 64 steps, a `pause` of at
most 10 s, 500 characters per `type` step — because it all runs in real time and the recording
is open the whole while. Split a long string across steps with the pauses a person would take.

Only the *choosing* of a target is different from a recording a person starts. The countdown
and the floating HUD still show, whoever began it: a recording an agent started is never
invisible at the machine it runs on. `countdownSeconds: 0` skips the wait, not the HUD.
Anything left out of `options` keeps whatever the user last chose in the app, and an agent's
choices apply to its own recording without saving over those settings.

The Screen Recording permission is the app's, not the agent's. Without it `start_recording`
fails with `screen_recording_denied` and Lumae shows its onboarding window, because an agent
cannot grant a macOS privacy permission on the user's behalf — ask the person at the machine.

One trap belongs to display and area capture in particular: Lumae's own windows are left out
of the *picture*, but they are still on the screen. An agent aiming by coordinate can click a
window that the recording does not show, which does nothing it can see, records no click, and
reports no error — measured 2026-09-19, a take that came back with none of its clicks. While
`selfCaptureAllowed` is false those windows are missing from `list_capture_targets` too, so
their frames have to come from somewhere else: move them aside first, or film a window rather
than a display.

Lumae's own windows are listed with `isLumae: true` but refuse to be captured unless the
`capture.allowsSelf` preference is on (`defaults write coderlife.Lumae capture.allowsSelf
-bool YES`); filming the editor is for making Lumae's own material, not something to stumble
into.

**A caveat worth knowing before reading frames back.** ScreenCaptureKit stores a frame only
where the picture changed, so a demo that sits still holds one frame for as long as nothing
moves — a still window measured six frames in fifteen seconds. `render_frame` with `source:
true` reads the recording itself, so it has nothing to show inside such a stretch and says so
in those words. The composed frame (the default) has one at every time.

**Marks, and the dead air problem.** A recording runs in real time while an agent thinks, so a
screenshot-reason-act loop leaves seconds of nothing between every action. `mark_moment` is the
cheap fix: called as each beat happens, it timestamps a label into the recording, and
`get_project` reports marks per clip in both timeline and media time alongside the clicks. The
silence between two marks is the dead air to cut, and the marks themselves say which beats are
worth a zoom. Nothing else in a recording records what a click *meant*.

**Cutting the take down.** A recording runs in real time, so it carries whatever the agent did
before its first action and after its last — and its own thinking in between. `trimToContent`
on `stop_recording` removes the ends: from half a second before the first mark to a settle after
the last thing the driver did. The two ends come from different places deliberately. A mark says
when a beat *started*, which is the right head and the wrong tail, since the beat is what follows
it — trimming to the last mark truncates the last thing the demo did. The tail is the latest of
the marks and the recorded input instead, because the dead air at the end is the agent doing
nothing, so the agent's own last action is what bounds it. A take with nothing worth removing is
left alone.

The recording's own frames were tried for the tail and dropped. A screen recording stores one
only where the picture changed, which sounds like the perfect signal until the subject is an app
that repaints continuously — Lumae's own glass does, and so do animations, carets and video.
Measured on a real take, six seconds of deliberate stillness still produced frames at 55 fps to
the last moment, so the signal said nothing on the recordings anybody would trim. If a trim
leaves more than you wanted, `edit_project` takes the rest off.

**Transcripts.** `transcribe` runs Apple's on-device speech recognition over the project's
narration (or the clips' own sound when there is none) and returns every word with the timeline
seconds it was said over. It changes nothing unless `writeCaptions` is passed, which groups the
words into caption lines and sets them as one undoable edit — reading a transcript to plan cuts
is a different job from wanting captions, and the common case should not be a mutation.

## How it works

The app hosts the server (the official MCP Swift SDK's `Server`) and listens
on a Unix socket inside its sandbox container. The helper relays the client's
stdio to that socket, launches the app when it is not running, and stages
files named by `import_media` into `~/Movies/.lumae-mcp-inbox` so the sandboxed
app is allowed to read them — that folder rather than the container beside the
socket, because only the app itself may create a file in its container. The app
removes each staged copy after importing, and clears anything a day old when it
starts. Only your user account can reach the socket.

## Troubleshooting

- **"Lumae is not available"** (JSON-RPC error -32000): the helper could not
  reach the app within 20 seconds. Check that Lumae launches on its own, and
  that the helper you configured is the one inside the Lumae you run
  (Settings › MCP Server shows the path).
- **"Lumae closed the connection"** (also -32000): Lumae quit, or an update
  relaunched it, while the call was in flight. Retry; the next request
  launches it again.
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
every tool asks for in its `context` argument. A `get_more_tools` call records
the agent's description of the gap in the same way, and nothing else. Never the
results or the project itself. Nothing about your projects leaves the Mac through MCP
except what the connected client asks for and shows you.
