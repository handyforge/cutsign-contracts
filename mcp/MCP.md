# CutSign over MCP

CutSign exposes the project that is **open in the running desktop app** to an MCP client (Claude Code, Codex, Cursor,
Gemini CLI, or any client that speaks MCP over stdio). The agent reads the timeline, looks at frames, edits through
the same commands the person's interface uses, and brings in files of its own (alpha video overlays, stickers,
subtitles). Every write is one Undo step for the person, and the timeline follows the agent's edits on screen.

The MCP server is a thin adapter started by the client: `CutSign.exe --mcp`. It holds no project state; it forwards
each tool call to the running app over a local pipe. If the app is not running, every call answers `app_not_running`.

For how an agent should work with the project (the loop, anchors, rendering overlays), see the skill:
[`docs/agent/cutsign-skill/SKILL.md`](agent/cutsign-skill/SKILL.md).

## What you need

- CutSign desktop, **running**, with the project open.
- A `CutSign.exe` at a **stable path** for the client to start: unpack `CutSign-<version>-win.zip` anywhere (for
  example `C:\Apps\CutSign\`). The examples below use `C:\Apps\CutSign\CutSign.exe`.
- **Not the portable exe** (`CutSign <version>.exe`). Its launcher starts the real app without passing stdin / stdout
  through, so a client sees nothing (measured 2026-10-27), and every launch empties and re-extracts one fixed folder
  under `%TEMP%`, then empties it again when that launch ends, which would pull files from under a copy already running
  from the same build. Run the app from the portable exe if you like; point the MCP client at the unzipped one. Both
  read the same profile, so the adapter finds the app either way.

## Connect

### Claude Code

```bash
claude mcp add cutsign -- "C:\Apps\CutSign\CutSign.exe" --mcp
# for every project on this machine:
claude mcp add --scope user cutsign -- "C:\Apps\CutSign\CutSign.exe" --mcp
```

### Codex (`~/.codex/config.toml`)

TOML literal strings (single quotes) keep the backslashes as they are.

```toml
[mcp_servers.cutsign]
command = 'C:\Apps\CutSign\CutSign.exe'
args = ['--mcp']
startup_timeout_sec = 30
```

### Cursor (`~/.cursor/mcp.json` or `.cursor/mcp.json` in a project)

```json
{
  "mcpServers": {
    "cutsign": { "command": "C:\\Apps\\CutSign\\CutSign.exe", "args": ["--mcp"] }
  }
}
```

### Gemini CLI (`~/.gemini/settings.json`)

```json
{
  "mcpServers": {
    "cutsign": { "command": "C:\\Apps\\CutSign\\CutSign.exe", "args": ["--mcp"] }
  }
}
```

### Strict clients: the Node-mode entry

`CutSign.exe --mcp` starts Electron, and the Electron binary itself writes **one empty line** (`\r\n`) to stdout before
any CutSign code runs; nothing in the app can stop it. Every client measured below skips that line. A client that
rejects any stdout line that is not JSON will not; give it the same adapter as a plain Node script, run by the same
exe with `ELECTRON_RUN_AS_NODE=1`, which writes nothing of its own:

```bash
claude mcp add cutsign -e ELECTRON_RUN_AS_NODE=1 -- "C:\Apps\CutSign\CutSign.exe" "C:\Apps\CutSign\resources\app.asar\out\main\mcp-adapter.js"
```

```toml
[mcp_servers.cutsign]
command = 'C:\Apps\CutSign\CutSign.exe'
args = ['C:\Apps\CutSign\resources\app.asar\out\main\mcp-adapter.js']
env = { ELECTRON_RUN_AS_NODE = "1" }
```

The variable must be in the client's configuration (`env`): MCP clients start servers with a small default
environment, not the one of your shell. Both entries serve the same tools from the same pipe.

### Another profile

The app and the adapter look for each other in the same profile folder. If you start the app with
`--user-data-dir=<folder>`, give the adapter the same switch: `CutSign.exe --mcp --user-data-dir=<folder>` (the Node-mode
entry takes it too).

### Development tree

After `npm run build` in `apps/desktop`, with the dev app started by `npm run dev` or `npm start`:

```bash
claude mcp add cutsign -- npx electron "<repo>\apps\desktop" --mcp
```

Start Electron on the **folder** `apps/desktop`, not on `out/main/main.js`: Electron names the profile after the
`package.json` it finds next to what it starts, so `electron out/main/main.js` reads `%APPDATA%\Electron` and never
finds the dev app (which uses `%APPDATA%\cutsign-desktop`), and it reports Electron's version as `appVersion`.

## Client compatibility (measured 2026-10-27, CutSign 0.7.0-beta.6, `electron-builder --win --dir` build)

| Client | `CutSign.exe --mcp` | Node-mode entry |
|---|---|---|
| MCP Inspector 2.12.0 CLI (`npx @modelcontextprotocol/inspector --cli … --method tools/list`) | pass: 27 tools, `tools/call` works | pass |
| Codex CLI 0.164.0 (`codex app-server`: `mcpServerStatus/list`, `mcpServer/tool/call`) | pass: 27 tools, call works | pass |
| Python MCP SDK 2.3.0 (`mcp.client.stdio`) | pass, with one logged parse error for the empty line | pass, no error |
| TypeScript MCP SDK 1.32.2 `Client` (the e2e tests and the packaged test) | pass, `onerror` once for the empty line | pass |
| Strict line reader (every stdout line must be JSON-RPC) | **fail** on the first line `"\r"` | pass |

Not measured: Cursor, Gemini CLI, and Claude Code itself (the owner's smoke test). If a client fails to start the server
with `CutSign.exe --mcp` and its log mentions a parse error on the first line, switch it to the Node-mode entry.

## How the adapter finds the app

The running app writes `<profile>\mcp\session.json` once its pipe listens, and removes it when it quits:

```json
{ "sessionId": "…", "pipe": "\\\\.\\pipe\\cutsign-<sessionId>-<32 hex>", "secret": "<64 hex>", "pid": 1234, "startedAt": "…" }
```

`<profile>` is `%APPDATA%\cutsign-desktop` for the packaged app and for the dev tree (Electron takes the folder name from
`package.json`; measured 2026-10-27), or the `--user-data-dir` folder.

- The adapter reads the file on its first call, connects, and sends the secret in its first frame. It reconnects on
  the next call after the app restarted (a new pipe and secret each run).
- **Stale file**: a crashed app cannot remove its file. A file whose `pid` is gone, or whose pipe no longer exists,
  answers `app_not_running` at once.
- **The app quits during a call**: the app sends `app:closing` (an MCP `notifications/message` log event), the call in
  flight fails with `app_not_running`, and so does every call until the app runs again.
- **One client at a time**: a second adapter connecting while one is attached gets `busy`, with the attached client's
  name in `data.clientName`.
- **Two app instances on one profile**: the newest app owns the session file, so a newly started adapter attaches to
  it. An older app keeps serving the adapter already attached to it, but is never written back into the file; when
  the newest quits, the file is removed and new adapters see `app_not_running` until an app starts again.
- **The client goes away**: when stdin closes, the adapter drops its pipe connection and exits with code 0 (measured
  under 50 ms).

## Security model

- **Local only.** The pipe is a Windows named pipe (`\\.\pipe\cutsign-<session>-<random>`), created with the default
  access of the user running the app; on macOS / Linux a Unix socket inside the profile. No TCP port, no HTTP.
- **A secret per run.** 64 random hex characters in the session file, inside the user's profile. The first frame on
  the pipe must carry it; anything else is answered `unauthorized` and closed, without a word about the session.
- **Footage paths stay in the app.** Tools name clips by id and file name, never by path.
- **The agent's files come from one place.** `get_workspace` returns the workspace folder: `.cutsign-agent\` next to
  the project file, or `<profile>\agent-workspace\<projectId>\` for a project without a file. Tools that take a path
  (`add_overlay_clip`, `import_overlay_manifest`, `import_subtitles`) accept only files under it, or under a brief
  folder `write_brief` made in this run, compared after resolving links (a junction pointing outside is refused).
  Limits: 1 GiB per overlay file or PNG folder, 4 MiB per subtitle file. The app references these files, it does not
  copy them.
- **Writes go through the interface's own commands**: the same validation, the same locked ranges, one Undo step each,
  recorded with the MCP client's id. Each write logs one audit line in the app's log (command, transaction,
  client), never the payload.
- **Bounded reads.** `list_segments` pages of 200, `get_timeline_view` windows of 600 s of source, `get_frame` at most
  640 px wide; a pipe frame is at most 2 MiB (a frame that would not fit is taken again at half the width).
- **Tools that do not exist, on purpose:** export, approving or rejecting review proposals, deleting or relinking
  sources, opening or closing projects, reading or writing arbitrary files, changing settings, applying an analysis,
  running programs. These stay with the person: they are irreversible, reach outside the open project, or are the
  person's decision. A tool description is documentation, never permission.

## Writing

- Every write takes **`expectedRevision`**: the revision you last read (`get_app_session`, `list_segments`, a previous
  write's result). If anyone wrote since, the answer is `revision_conflict` with the current revision in
  `data.revision`: read again, then decide again.
- **`requestId`** (optional, ≤ 100 characters) makes a write idempotent: a retry with the same id returns the first
  answer, and the same note / clip ids.
- A write answers `{ revision, previousRevision, transactionId, affected, warnings }`. `affected` lists
  `{ type, id }` with `type` one of `asset`, `segment`, `group`, `note`, `overlay`.
- `undo_agent_edit { transactionId }` undoes one of **your own** writes while it is still the newest undo step;
  another client's or the person's answers `not_own_transaction`.
- The person sees a burst of writes one at a time: every write commits at once, but the timeline moves to each in
  commit order about 0.7 s apart, with a chip "Agent · n edits behind" (more than 8 behind skips to the newest).
  A `focus_view { label }` just before a write names that step on the chip.

## Errors

A failed tool call is an MCP tool result with `isError: true` and one text item, `{ "code", "message", "data"? }`.
Branch on `code`; `message` is English for logs.

| Code | Meaning |
|---|---|
| `app_not_running` | No app is running for this profile, its session file is stale, or it closed during the call |
| `no_project` | The app runs with no project open |
| `busy` | Another MCP client is attached (`data.clientName`) |
| `unauthorized` | Wrong secret (only seen by something that is not the adapter) |
| `oversized`, `malformed` | A pipe frame over 2 MiB, or not one JSON object |
| `unknown_method`, `invalid_params` | No such tool; params that fail the tool's schema (unknown keys too) |
| `invalid_target` | An id the project does not have, or a time outside the clip or the output |
| `revision_conflict` | Someone wrote since your `expectedRevision`, or a `list_segments` cursor of an older revision (`data.revision`) |
| `not_own_transaction` | `undo_agent_edit` on a transaction that is not yours |
| `backend_failed` | FFmpeg or the backend failed (missing source file, undecodable frame) |
| `internal` | Anything else; the message says what |
| `overlay.path_outside_workspace` | A path outside the workspace and this run's brief folders |
| `overlay.file_too_large` | Over 1 GiB |
| `overlay.file_missing` | The file is not there |
| `overlay.unsupported_kind` | Not `.webm`, `.mov`, a PNG folder, `.png` or `.webp` |
| `overlay.no_alpha` | A video without alpha; pass `importAnyway: true` to add it anyway |
| `overlay.size_mismatch` | Frame size more than 1 % off the project's (`data.width`, `data.height`) |
| `overlay.project_not_exportable` | An `output` anchor (or `write_brief`) while segments are still pending |
| `overlay.project_mismatch`, `overlay.manifest_invalid` | A manifest made for another project, or not a manifest |
| `overlay.probe_failed`, `overlay.convert_failed` | The file could not be read as video, or the PNG frames could not be joined |
| `job.busy` | An export or another import is running (`data.task`) |
| `command.locked_range` | The range is locked by the person |
| `command.invalid_result` | The edit would break the project (for example an id seed already used) |

## Tool reference

Generated from `apps/desktop/shared/mcp/tools.ts` and `write-tools.ts` by `npm run mcp:doc` (in `apps/desktop`); a unit
test fails when this section is out of date.

<!-- BEGIN GENERATED TOOL REFERENCE: scripts/mcp-tools-doc.mjs, do not edit by hand -->

27 tools. Every write takes `expectedRevision` (required) and `requestId` (optional); see "Writing" above.

| Tool | Kind | Title |
|---|---|---|
| [`get_app_session`](#get_app_session) | read | App session |
| [`get_workspace`](#get_workspace) | read | Agent workspace |
| [`get_project_summary`](#get_project_summary) | read | Project summary |
| [`list_segments`](#list_segments) | read | List segments |
| [`get_timeline_view`](#get_timeline_view) | read | Timeline window |
| [`get_output_map`](#get_output_map) | read | Output map |
| [`list_notes`](#list_notes) | read | List notes |
| [`list_overlay_clips`](#list_overlay_clips) | read | List overlay clips |
| [`get_frame`](#get_frame) | read | Get frame |
| [`focus_view`](#focus_view) | write | Focus view |
| [`split_segment`](#split_segment) | write | Split segment |
| [`set_range_included`](#set_range_included) | write | Include or cut a range |
| [`trim_segment`](#trim_segment) | write | Trim segment |
| [`move_asset`](#move_asset) | write | Move clip |
| [`set_editor_scope`](#set_editor_scope) | write | Editor scope |
| [`propose_range`](#propose_range) | write | Propose a cut |
| [`add_note`](#add_note) | write | Add note |
| [`add_notes`](#add_notes) | write | Add notes |
| [`add_redaction`](#add_redaction) | write | Add redaction |
| [`update_overlay_clip`](#update_overlay_clip) | write | Update overlay clip |
| [`remove_overlay_clip`](#remove_overlay_clip) | write | Remove overlay clip |
| [`undo_agent_edit`](#undo_agent_edit) | write | Undo my edit |
| [`add_overlay_clip`](#add_overlay_clip) | write | Add overlay clip |
| [`write_brief`](#write_brief) | write | Write overlay brief |
| [`import_overlay_manifest`](#import_overlay_manifest) | write | Import overlay manifest |
| [`import_subtitles`](#import_subtitles) | write | Import subtitles |
| [`set_subtitles`](#set_subtitles) | write | Subtitle export settings |

### get_app_session

**App session.** The running CutSign app: session id, open project id, its revision, project type (preset) and app version. Call this first.

`readOnlyHint: true` · `openWorldHint: false`

No parameters.

### get_workspace

**Agent workspace.** The folder where the agent may write files it brings into the project (renders, subtitles). Created on demand.

`readOnlyHint: true` · `openWorldHint: false`

No parameters.

### get_project_summary

**Project summary.** Clips (id, name, duration in pts, timebase, status), groups, review counts and the editor scope of the open project.

`readOnlyHint: true` · `openWorldHint: false`

No parameters.

### list_segments

**List segments.** Segments in timeline order, filtered by clip, group, review state, origin or a clip-local pts window. Pages of at most 200.

`readOnlyHint: true` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | no | Only segments of this clip. Required when from / to are given (positions are clip-local). |
| `groupId` | string (≤ 200 chars) | no | Only segments of clips in this group. |
| `review` | "pending" \| "approved" | no |  |
| `origin` | string (≤ 40 chars) | no | analysis-keep, ai, manual, silence, speech, loud, ... |
| `included` | boolean | no |  |
| `from` | integer ≥ 0 | no | Clip-local pts: only segments ending after it. |
| `to` | integer ≥ 0 | no | Clip-local pts: only segments starting before it. |
| `cursor` | string (≤ 80 chars) | no | nextCursor of the previous page, valid on the same revision only. |
| `limit` | integer ≥ 1, ≤ 200 | no |  |

### get_timeline_view

**Timeline window.** Every segment overlapping a window of one clip, with the revision; the structured view to verify an edit against.

`readOnlyHint: true` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `sourceInPts` | integer ≥ 0 | yes |  |
| `sourceOutPts` | integer ≥ 0 | yes | Window end; the window may span at most 600 s of source. |

### get_output_map

**Output map.** Which stretch of which clip plays at which second of the export: frame size, fps and the pieces in output order.

`readOnlyHint: true` · `openWorldHint: false`

No parameters.

### list_notes

**List notes.** Notes of the project (text anchored on a clip-local pts with a duration), optionally of one clip.

`readOnlyHint: true` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | no |  |

### list_overlay_clips

**List overlay clips.** Overlay clips (alpha videos and stickers) with their anchor and whether they still match the timeline.

`readOnlyHint: true` · `openWorldHint: false`

No parameters.

### get_frame

**Get frame.** One JPEG frame of the source at a clip-local pts, or at a second of the export. Overlays and stickers are not drawn on it.

`readOnlyHint: true` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | no | With pts: a frame of this clip. |
| `pts` | integer ≥ 0 | no | Clip-local pts, in the clip timebase. |
| `outputSec` | number ≥ 0 | no | Instead of assetId + pts: a second of the exported sequence. |
| `maxWidth` | integer ≥ 64, ≤ 640 | no | Width of the JPEG, at most 640 (default 480). |

### focus_view

**Focus view.** Points the open timeline at a range of one clip, without editing. Not an undo step.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `inPts` | integer ≥ 0 | yes |  |
| `outPts` | integer ≥ 0 | yes |  |
| `label` | string (≤ 80 chars) | no | A short chip shown over the range. |

### split_segment

**Split segment.** Splits the segment under a clip-local pts in two. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `pts` | integer ≥ 0 | yes |  |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### set_range_included

**Include or cut a range.** Keeps (included: true) or cuts a clip-local pts range. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `inPts` | integer ≥ 0 | yes |  |
| `outPts` | integer ≥ 0 | yes |  |
| `included` | boolean | yes |  |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### trim_segment

**Trim segment.** Moves the in or out edge of a segment to a pts (clamped to its neighbours). Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `segmentId` | string (≤ 200 chars) | yes |  |
| `edge` | "in" \| "out" | yes |  |
| `pts` | integer ≥ 0 | yes |  |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### move_asset

**Move clip.** Moves a clip to another position of the sequence. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `targetIndex` | integer ≥ 0 | yes |  |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### set_editor_scope

**Editor scope.** Works on one clip alone, or on the whole sequence (null). Not an undo step.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `isolatedAssetId` | string (≤ 200 chars) \| null | yes | A clip to work on alone, or null for the whole sequence. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### propose_range

**Propose a cut.** A pending cut proposal over a range, for the person to keep or drop. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `inPts` | integer ≥ 0 | yes |  |
| `outPts` | integer ≥ 0 | yes |  |
| `source` | { kind: "assistant", comment, index, model, measurements? } | yes |  |
| `reasonCode` | string `^[a-z][a-z0-9_]{0,63}$` | yes |  |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### add_note

**Add note.** One note (a line of text) at a clip-local pts; durationPts > 0 makes it a subtitle line. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `pts` | integer ≥ 0 | yes |  |
| `durationPts` | integer ≥ 0 | yes |  |
| `text` | string (≤ 500 chars) | yes |  |
| `source` | { kind: "assistant", comment, index, model, measurements? } | no | Provenance shown on the proposal card: the comment it answers, its index and the model name. |
| `id` | string `^[A-Za-z0-9_-]{1,80}$` | no | An existing note id: replaces its text. |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### add_notes

**Add notes.** Up to 500 notes in one undo step (subtitles, captions). Answers the result and noteIds in order.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lines` | array of { assetId, pts, durationPts, text } (≥ 1, ≤ 500 items) | yes |  |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### add_redaction

**Add redaction.** A pixelate / blur / solid box over a range of a clip. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `assetId` | string (≤ 200 chars) | yes |  |
| `inPts` | integer ≥ 0 | yes |  |
| `outPts` | integer ≥ 0 | yes |  |
| `rect` | { x, y, w, h } | yes | A box in fractions of the frame: 0 <= x, y; w, h > 0; x + w <= 1; y + h <= 1. |
| `style` | "pixelate" \| "blur" \| "solid" | yes |  |
| `source` | { kind: "assistant", comment, index, model, measurements? } | no | Provenance shown on the proposal card: the comment it answers, its index and the model name. |
| `needsPlacement` | boolean | no |  |
| `idSeed` | string `^[A-Za-z0-9_-]{1,64}$` | no | Makes the ids this call creates predictable (letters, digits, - and _). Default: derived from requestId. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### update_overlay_clip

**Update overlay clip.** Switches, relabels, re-anchors or moves an overlay clip. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string (≤ 200 chars) | yes |  |
| `enabled` | boolean | no |  |
| `label` | string (≤ 200 chars) | no |  |
| `acceptedMapHash` | string | no |  |
| `anchor` | { kind: "source", assetId, pts, durationPts } \| { kind: "output", startSec, endSec, revision, outputMapHash } | no |  |
| `placement` | { x, y, w, h } | no | A box in fractions of the frame: 0 <= x, y; w, h > 0; x + w <= 1; y + h <= 1. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### remove_overlay_clip

**Remove overlay clip.** Takes an overlay clip off the project; its file stays on disk. Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string (≤ 200 chars) | yes |  |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### undo_agent_edit

**Undo my edit.** Undoes one of your own writes by transactionId, when it is still the newest undoable step. Never another client's.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `transactionId` | string (≤ 200 chars) | yes | A transactionId one of your own writes returned. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |

### add_overlay_clip

**Add overlay clip.** Brings one file of yours into the project as an overlay: an alpha video (animation, video overlay, styled text) or a still sticker. The file must be under the workspace and at most 1 GiB. Anchor on a clip (source pts) or on output seconds (needs an exportable project). Answers clipId, status, width, height, fps, durationSec, hasAlpha.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | string (≤ 4096 chars) | yes | A file under the workspace (get_workspace): .webm (VP9 alpha), .mov (ProRes 4444), a folder of PNG frames, or a still .png / .webp. |
| `anchor` | { kind: "source", assetId, pts, durationPts? } \| { kind: "output", startSec, endSec } | yes |  |
| `label` | string (≤ 200 chars) | no |  |
| `placement` | { x, y, w, h } | no | Still images only: where it sits. Default: bottom-right in the safe area. |
| `importAnyway` | boolean | no | Add a video without an alpha channel anyway (it will cover the picture). |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### write_brief

**Write overlay brief.** Writes a brief folder into the workspace (brief-<revision>/): BRIEF.md, AGENTS.md, cutsign-brief.json, the output map, proxy and thumbnails. Read AGENTS.md for the render recipe. Needs an exportable project.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `textOnly` | boolean | no | Skip the 720p proxy and the thumbnails (fast; the map and the notes only). |
| `request` | string (≤ 2000 chars) | no | What the overlay should do, written into BRIEF.md. |
| `language` | "en" \| "vi" | no | Language of BRIEF.md (default en). |

### import_overlay_manifest

**Import overlay manifest.** Imports a cutsign-overlays.json (the brief's manifest) in one undo step. Answers clipIds and warnings for the clips left out.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `manifestPath` | string (≤ 4096 chars) | yes | cutsign-overlays.json under the workspace or a brief folder of this run. |
| `importAnyway` | boolean | no |  |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### import_subtitles

**Import subtitles.** Reads a .srt or .vtt of yours into notes on one clip, in one undo step: each cue becomes a note with a duration, which the export writes as a soft subtitle track. Cue times are seconds of that clip; to time them on the export, convert with get_output_map first. At most 500 cues, text cut to 500 characters, overlapping cues kept. Answers the result, noteIds and skipped (empty cues, zero-length cues and cues past the clip's end).

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `file` | string (≤ 4096 chars) | yes | A .srt or .vtt file under the workspace (get_workspace) or a brief folder of this run. |
| `assetId` | string (≤ 200 chars) | yes | The clip the cue times belong to: they are seconds of that clip (clip-local), not of the export. |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

### set_subtitles

**Subtitle export settings.** Whether a video export writes the timed notes as a soft subtitle track (soft) and as an .srt beside the file (sidecar). Answers the command result: revision, transactionId, affected.

`readOnlyHint: false` · `destructiveHint: false` · `openWorldHint: false`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `soft` | boolean | no | Write the notes that have a duration as a subtitle track inside the exported MP4 (default on). |
| `sidecar` | boolean | no | Also write <output name>.srt next to the exported file (default off). |
| `expectedRevision` | integer ≥ 0 | yes | The revision you read (get_app_session, list_segments, ...). Another writer since then answers revision_conflict. |
| `requestId` | string (≤ 100 chars) | no | Idempotency key: a retry with the same id returns the first answer. |

<!-- END GENERATED TOOL REFERENCE -->
