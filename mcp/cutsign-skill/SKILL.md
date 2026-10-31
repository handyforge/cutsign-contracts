---
name: cutsign
description: Edit a video project in a running CutSign desktop app over MCP — read the timeline, look at frames, cut, add notes and subtitles, bring your own overlays (stickers, animations, alpha video). The person keeps the final say; every edit is one Undo step.
---

# CutSign — working with the open project

CutSign is a desktop editor whose project is one JSON file (`.cutsign.json`, schema at
https://github.com/handyforge/cutsign-contracts). While the app is running, it exposes an MCP server: you attach to the
**open project**, read it, edit it through the same commands the person's UI uses, and the timeline follows your edits
on screen. You never write the project file yourself and never touch the person's footage.

## Connect

```bash
# the unzipped app (CutSign-<version>-win.zip), not the portable CutSign <version>.exe
claude mcp add cutsign -- "C:\Apps\CutSign\CutSign.exe" --mcp
# developer tree, after npm run build in apps/desktop
claude mcp add cutsign -- npx electron "<repo>\apps\desktop" --mcp
```

The adapter finds the running app through a session file in the user's profile. If the app is not running, every
call returns `app_not_running`; with the app running but no project open, `no_project`. Tell the person to open the
project; do not try to start the app or open files yourself. If the app quits while you work, you get `app_not_running`
(and an `app:closing` log notification): stop writing and tell the person what was and was not applied.

### Codex, Cursor, Gemini CLI

Same server, configured in the client's own file (`~/.codex/config.toml`, `~/.cursor/mcp.json`,
`~/.gemini/settings.json`). The exact snippets, the Node-mode entry for clients that reject a blank stdout line, the
security model and the generated tool reference are in
[`docs/MCP.md`](https://github.com/handyforge/cutsign/blob/main/docs/MCP.md).

## The loop

1. **Orient:** `get_app_session` (project id, `revision`, `preset`) → `get_project_summary` (clips, durations, review
   counts) → `get_output_map` (what plays when in the export).
2. **Read before you write:** `list_segments` with a clip id and a pts window (pages of 200; pass `nextCursor` back),
   or `get_timeline_view` for the exact window you are about to change (≤ 600 s of source). Positions are
   **clip-local integer pts** in the clip's `timebase` (normally milliseconds). Never guess a position from a file
   name or a rough second: read the segment ids and pts first.
3. **Look:** `get_frame { assetId, pts }` or `get_frame { outputSec }` returns a JPEG (≤ 640 px, `maxWidth` to ask for
   less). Use it to check a cut point, a sticker placement, or what a silence actually contains. Overlays are not drawn
   on it.
4. **Write one thing at a time**, always passing the `expectedRevision` you last read, and a `requestId` of your own
   if you may retry (a retry with the same id returns the first answer and the same new ids). The result carries the
   new `revision` (use it for the next write), the `transactionId`, and `affected` — `{ type, id }` items, `type` one of
   `asset`, `segment`, `group`, `note`, `overlay`. `revision_conflict` means the person (or another agent) edited in
   between; `data.revision` is the current one: re-read the window, then decide again. Every write is one Undo step
   for the person; `undo_agent_edit { transactionId }` reverts **your own** transaction while it is still the newest
   step (someone else's answers `not_own_transaction`).
5. **Verify:** `get_timeline_view` on the same window, and `get_frame` where it matters.
6. **Stop at the decision:** proposals (`propose_range`) stay `pending` until the person approves them in the app. Do
   not try to approve, export, delete sources, or change settings — those tools do not exist on purpose.

### What the person sees while you write

Each write commits at once — Undo, export and your next read are always current — but when you fire several in a row
the timeline shows them one at a time, in commit order, about 0.7 s each, with a chip "Agent · n edits behind" beside
the toast. More than 8 behind and it jumps to the newest. Esc or a click on the timeline pauses the pacing; that is the
person taking the view back, not an error. To name a step on the chip, call `focus_view { assetId, inPts, outPts,
label }` just before that write. The toast's Undo takes the whole burst back.

## What you may change

Every write below also takes `expectedRevision` (required) and `requestId` (optional).

| Want | Tool | Notes |
|---|---|---|
| Cut / keep a range | `set_range_included { assetId, inPts, outPts, included }` | Locked ranges refuse with `command.locked_range` |
| Split at a point | `split_segment { assetId, pts }` | Splits the segment under `pts` |
| Trim an edge | `trim_segment { segmentId, edge: "in" \| "out", pts }` | Clamped to the neighbours |
| Reorder clips | `move_asset { assetId, targetIndex }` | |
| Work on one clip alone | `set_editor_scope { isolatedAssetId }` | `null` for the whole sequence |
| Suggest a cut for review | `propose_range { assetId, inPts, outPts, source: { kind: "assistant", comment, index, model }, reasonCode }` | Shows as a pending card with your model name |
| A note at a time | `add_note { assetId, pts, durationPts, text }` | `durationPts > 0` makes it a subtitle line; notes also become EDL comments |
| Many lines at once (captions, chapter notes) | `add_notes { lines: [{ assetId, pts, durationPts, text }] }` | ≤ 500 lines, text ≤ 500 characters, one Undo step; answers `noteIds` in order |
| Fix a note's words, time or length | `update_note { id, text?, pts?, durationPts? }` | At least one field; the note stays inside its clip (`command.invalid_result` otherwise); ids from `list_notes` |
| Delete a note | `remove_note { id }` | An attached shelf asset is detached (the image stays on the shelf); an overlay clip made from it (`fromNoteId`) stays |
| Subtitles from a file | `import_subtitles { file, assetId }` | See below |
| Subtitle export switches | `set_subtitles { soft?, sidecar? }` | Soft track in the MP4 (default on), `.srt` beside it (default off) |
| Hide something | `add_redaction { assetId, inPts, outPts, rect, style }` | `rect` is fractions of the frame; `style` `pixelate` / `blur` / `solid`; the person confirms placement before export |
| Sticker (PNG / WebP) | `add_overlay_clip { file, anchor, placement? }` | Goes to the person's "My shelf" (not copied); `anchor` must be `source` (default 3 s) |
| Animation / lower third / styled text / video overlay | `add_overlay_clip { file, anchor, importAnyway? }` | `file` is a VP9 **alpha** WebM, ProRes 4444 MOV, or a folder of PNG frames you rendered; same frame size as the project (1 %) |
| Change or remove an overlay | `update_overlay_clip { id, enabled?, label?, anchor?, placement? }`, `remove_overlay_clip { id }` | Removing leaves the file on disk |
| Point the person at a spot | `focus_view { assetId, inPts, outPts, label }` | Not an edit; no revision |

`add_overlay_clip` answers `clipId`, `status`, `width`, `height`, `fps`, `durationSec`, `hasAlpha`. A video without
alpha is refused with `overlay.no_alpha`; pass `importAnyway: true` only when covering the picture is what the person
asked for.

### Anchors

- `{ kind: "source", assetId, pts, durationPts }` — glued to one clip's footage; a cut inside the range cuts the
  overlay with it. Use this for stickers and anything that belongs to a shot. `durationPts` is required for a video.
- `{ kind: "output", startSec, endSec }` — seconds of the export, for a clip that spans a join. Only accepted when the
  project can already export (no pending segments, `overlay.project_not_exportable` otherwise); then use a `source`
  anchor.

### Subtitles

`import_subtitles { file, assetId }` reads a `.srt` or `.vtt` from the workspace into notes on **one clip**, in one Undo
step. Cue times are **seconds of that clip** (clip-local), not of the export: to time a subtitle on the export, convert
with `get_output_map` first. At most 500 cues and 4 MiB; text is cut to 500 characters; overlapping cues are kept. The
answer has `noteIds` and `skipped` — the number of cues left out (empty after removing markup, zero length, or starting
at or after the clip's end; a cue running past the end is clipped, not skipped). A file with no usable cue is
`invalid_params` and writes nothing. Notes with a duration become a soft subtitle track when the person exports.

## Your files: the workspace

`get_workspace` returns `{ dir, briefDirs }`. **Every file you bring into the project must be under `dir`** (or under a
brief folder the app wrote for you in this run); other paths are refused with `overlay.path_outside_workspace`, and a
link pointing outside counts as outside. Put renders in `dir/renders/`, subtitles in `dir/subtitles/`. Overlay files are
at most 1 GiB. The app references these files, it does not copy them: do not delete or move them afterwards.

## Rendering an overlay (HyperFrames)

Ask for a brief first: `write_brief { request?, language?, textOnly? }` writes `dir/brief-<revision>/` with `BRIEF.md`
(your `request` goes in it, ≤ 2 000 characters; `language` `en` or `vi`), `AGENTS.md`, a 720p proxy, thumbnails,
`frame.md` (size, fps, safe areas) and `tokens.css` (the project's colours and fonts). `textOnly: true` skips the proxy
and thumbnails. It needs an exportable project. Follow `AGENTS.md` exactly: it holds the render command, the required
codec (VP9 with alpha), the frame size and fps, and the manifest format. Then either
`import_overlay_manifest { manifestPath, importAnyway? }` for the whole set (one Undo step; answers `clipIds` and
`warnings` for clips left out) or `add_overlay_clip` per file.

An MP4 has no alpha and is refused. Renders must match the project frame size (1 % tolerance, `overlay.size_mismatch`).

## Rules of the house

- Read the window before every write; pass `expectedRevision`; verify after.
- Prefer `propose_range` over `set_range_included` when the person has not told you to cut outright.
- Keep text in notes short and in the project's language; no markdown in note text.
- Do not loop over hundreds of single calls where a bulk tool exists (`add_notes`, `import_subtitles`,
  `import_overlay_manifest`).
- When a tool returns `command.locked_range`, `invalid_target`, `job.busy` (an export is running) or `busy` (another
  agent is attached), report it to the person; do not work around it.
- Use `undo_agent_edit` only on your own transactions, and only to take back your own mistake.
