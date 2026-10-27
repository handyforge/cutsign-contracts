# CutSign project file (`.cutsign.json`) — specification for outside readers and writers

**Version of this text:** 1 (2026-10-27) · **Schema:** [`project.v4.schema.json`](./project.v4.schema.json) (JSON Schema
2020-12, `$id https://handyforge.app/cutsign/contracts/project-file/project.v4.schema.json`) · **Fixtures:**
[`fixtures/`](./fixtures/) · **Change rules:** [`../README.md`](../README.md) §Change rules (restated in §12 below).

This document is written for someone who has never seen the app: a script, an MCP tool, an AI agent, or another editor that
wants to read or write a CutSign project. The schema is the arbiter; this text explains what the fields mean, what the
readers do with them, and what we promise about stability. Where the two disagree, the schema wins and this text has a bug.

## 1. What a project file is

- One UTF-8 JSON document, extension `.cutsign.json`. Older extensions `.footagebuild.json` and `.muxpro.json` still open
  (same content, older names).
- It describes **an edit over source videos that are never modified**: which files, in which order, which time ranges stay
  in the output, who proposed each range, whether a person approved it, and how the output is presented (notes, boxes,
  overlays, joins).
- It references media by absolute path (`assets[].path`, `presentation.overlayClips[].file`). Nothing is embedded.
- The app autosaves the open project to `autosave.cutsign.json` next to the project; the project the user opened is only
  rewritten on an explicit save.

## 2. Stability promise

*(decided 2026-10-27, BRD `OPEN_EDITOR_FOR_CREATORS_AND_AGENTS`)*

1. **The schema major version changes only with the app major version.** Within an app major, the document shape does not
   change: no required field is added, no field is removed or retyped, no enum value is removed.
2. **Optional fields may be added within a major.** Every reader ignores fields it does not know *except* that the schema
   uses `additionalProperties: false`, so an unknown field makes the document invalid for the schema of that app version.
   A writer should emit only fields listed in the schema it targets.
3. **The app reads every earlier version it ever wrote** (today: 2, 3, 4) and upgrades it on open. A document with a
   `version` the app does not know is refused with one coded error, never silently repaired:
   `[project.version_unsupported] Project version {n} was saved by a newer CutSign. Update the app to open it.`
4. **Enum values the app writes are listed in the schema.** A new value is added to the schema in the same change that
   starts writing it.
5. **Fixtures are the contract's test vectors.** Every fixture under `fixtures/` opens in the desktop, exports in the Rust
   backend, and validates in the Python engine, in the test suites of all three.

## 3. Versions

| `version` | Shape | Written by | Status |
|---|---|---|---|
| 1 | engine `analyze` payload, no `assets` key (`analysis.v1.schema.json`) | engine | read only, converted on import |
| 2, 3 | same shape, no `preset` / `presentation` | older app / CLI | read, upgraded to 4 on open with `preset: "cctv"` |
| **4** | current; adds `preset` (required), `presentation`, `silenceThreshold`, `asset.role`, `segment.audio`, origins `silence` / `speech` / `loud` | desktop, CLI `--pre` | **write this** |

A reader tells 1 from 2+ by the presence of `assets`.

## 4. Document layout

| Key | Req. | Meaning |
|---|---|---|
| `version` | yes | `4` for a new document |
| `preset` | yes (v4) | `creator` or `cctv`: which analysis signals run and whether detected people lock footage. See §5 |
| `id` | no | UUID the desktop assigns once; keep it when rewriting a file, omit it when creating one |
| `createdAt` | yes | ISO-8601 timestamp (see `$defs/timestamp` for the accepted pattern) |
| `reviewedAt` | no | last human review edit, desktop only |
| `assets[]` | yes | source videos, §6 |
| `sourceOrder[]` | yes | asset ids in sequence order; ids without an asset are skipped by the exporter |
| `manualOrder` | no | `true` once the user reordered by hand; stops automatic chronological sorting |
| `segments[]` | yes | time ranges over assets, §7 |
| `groups[]`, `groupingConfig` | no | media-pool grouping, §8; the desktop rebuilds unpinned groups on open |
| `silenceThreshold` | no | `{ levelDb ≤ 0, seconds ≥ 0 }` for preset `creator`; absent = engine default |
| `presentation` | no | notes, redactions, overlay clips, joins, audio, speed, look, §9 |
| `preprocessingRuns[]` | no | history of analysis / auto-cut runs; informational |
| `editor` | no | last view: `{ mode: "sequence" \| "isolated", isolatedAssetId }` |

### 4.1 Minimal valid v4 document

```json
{
  "version": 4,
  "preset": "creator",
  "createdAt": "2026-10-27T09:00:00Z",
  "assets": [
    {
      "id": "asset-1b2c3d4e",
      "path": "C:/footage/talk.mp4",
      "name": "talk.mp4",
      "capturedAt": "2026-10-27T08:00:00Z",
      "capturedAtSource": "filesystem",
      "importOrder": 0,
      "durationPts": 26000,
      "timebase": { "num": 1, "den": 1000 },
      "status": "ready"
    }
  ],
  "sourceOrder": ["asset-1b2c3d4e"],
  "segments": [
    { "id": "seg-1", "assetId": "asset-1b2c3d4e", "sourceInPts": 0,     "sourceOutPts": 12000, "included": true,  "origin": "manual",  "review": "approved" },
    { "id": "seg-2", "assetId": "asset-1b2c3d4e", "sourceInPts": 12000, "sourceOutPts": 14000, "included": false, "origin": "silence", "review": "approved", "audio": { "levelDb": -52.3 } },
    { "id": "seg-3", "assetId": "asset-1b2c3d4e", "sourceInPts": 14000, "sourceOutPts": 26000, "included": true,  "origin": "manual",  "review": "approved" }
  ]
}
```

This opens in the desktop and exports (one output of 24 s, the silence removed). `fixtures/desktop.v4.json` is the
exhaustive positive example: every optional field of v4 appears there once.

## 5. Preset

| `preset` | Signals the engine runs | Locking | Default |
|---|---|---|---|
| `creator` | black frames, **silence / speech / loud** from audio (`silenceThreshold`) | nothing locks; a file without audio is not an error | **the app default for a new project** *(decided 2026-10-27)*; the CLI default |
| `cctv` | black frames, people detection | a segment with origin `human-protected` is locked for every reader | what version 2 / 3 documents become on open |

The preset is chosen when the project is created. Changing it later means re-analysis (the desktop offers one path,
"Change and re-analyze"). The Rust exporter and the engine validate the value but do not branch on it.

## 6. Assets

Required: `id`, `path`, `name`, `capturedAt`, `capturedAtSource`, `importOrder`, `durationPts`, `timebase`, `status`.

- **Time unit.** Every `…Pts` integer is in units of `timebase` seconds (`num/den`). The desktop and the CLI always write
  `1/1000`, i.e. milliseconds. A writer may use another positive rational; the exporter rejects non-positive values.
- **`id`.** `asset-` + FNV-1a 32-bit hash (over UTF-16 code units, base 36) of the lower-cased path with `/` separators.
  `fixtures/asset-id.json` pins the function with inputs that include non-ASCII and non-BMP characters. The schema only
  requires a non-empty string that `segments[].assetId` and `sourceOrder[]` point at; use the derivation above so that
  re-importing the same file matches the asset the app would create. Whether the desktop tolerates a foreign id on open is
  not part of the contract.
- **`status`.** `ready` is the only state the exporter renders. `queued` (duration 0, not probed) and `analyzing` are
  desktop transients; `error` carries `error` text.
- **`capturedAt` / `capturedAtSource`.** Chronological ordering and date grouping use `capturedAt`; the source enum says
  where it came from (`embedded`, `filename`, `filesystem`, `import-order`, `manual`).
- Optional probe data: `width`, `height`, `fps`, `sourceFingerprint { size, modifiedNs }`, `relativeSourceFolder`,
  `minCutSeconds`, `cameraLabel`, `role` (`camera` default, `audio`, `screen`).

## 7. Segments

Required: `id`, `assetId`, `sourceInPts`, `sourceOutPts` (exclusive end), `included`, `origin`, `review`.

- **Coverage.** Segments of one asset are expected to tile the asset's source time in order; the exporter drops a piece
  whose end is not after its start or runs past `durationPts`. Adjacent included pieces are merged on export (rule in
  `../export-pieces/`).
- **`origin`** says who proposed the range:

  | Value | Meaning |
  |---|---|
  | `manual` | a person cut it |
  | `ai` | an automatic proposal (engine or in-app Assistant); with `source` when the Assistant made it |
  | `analysis-keep` | engine: keep |
  | `black` | engine: black frames |
  | `human-protected` | engine (`cctv`): a person is visible; **locked for every reader regardless of `locked`** |
  | `silence`, `speech`, `loud` | engine (`creator`): audio classes; `audio.levelDb` carries the quietest / peak dBFS |
  | `unclassified` | engine: no signal |

- **`review`.** `pending` or `approved`. **The exporter refuses a project that has any pending segment.** A tool that
  proposes a cut should write `pending` and leave approval to the person; a tool that acts on the person's explicit
  decision may write `approved`.
- **`locked`.** Optional, default false; readers OR it with `origin == "human-protected"`.
- **`source`** (`$defs/proposalSource`): provenance of an Assistant proposal: `{ kind: "assistant", comment, index, model,
  measurements? }`. An outside agent that proposes segments SHOULD fill it (its `model` name, the instruction it acted on in
  `comment` ≤ 500 chars) so the card shows who asked. Readers never derive behaviour from it.
- **`label`.** Free text for humans; the desktop builds card text from `origin` + `audio`, never from `label`.

## 8. Groups

`groups[]` (`id`, `name`, `kind`, `assetIds[]`, `order`, `collapsed`, `pinnedMembership`) and `groupingConfig` (`mode`,
`timezone`, `sessionGapMinutes?`, `filenamePattern?`). Only groups with `pinnedMembership: true` (manual sections)
survive a round trip; the desktop rebuilds the rest from `groupingConfig` on open. A writer that does not care about the
media pool may omit both.

## 9. Presentation

All optional; absent means "nothing".

| Field | Meaning | Who acts on it |
|---|---|---|
| `notes[]` | `{ assetId, pts, durationPts, text, id?, source?, assetRef? }`: reviewer note at a source time; also the soft-subtitle source and `* COMMENT:` lines in the EDL | desktop, Rust (EDL) |
| `redactions[]` | `{ assetId, inPts, outPts, rect, style, id?, source?, needsPlacement? }`: box burned into the export; `rect` is fractions of the frame; **any `needsPlacement: true` blocks export** | desktop, Rust |
| `overlayClips[]` | alpha videos (`kind` absent / `video`, full frame) or shelf stickers (`kind: "image"`, needs `placement` + `assetRef`) laid over the output; anchored to a clip (`anchor.kind: "source"`) or to output seconds (`anchor.kind: "output"`, pinned to an output-map hash) | desktop, Rust |
| `joints[]` | how the output joins after a segment: `cut` or `dip-black` with `durationMs`, optional gap caption | desktop, Rust |
| `overlays` | burn-in switches `timestamp`, `cameraLabel`, `titleCard` (`{ enabled }`) | desktop |
| `audio` | `original` (default), `mute`, `normalize` | Rust |
| `speed[]` | `{ segmentId, factor > 0 }` | reserved |
| `look` | `null` or `{}` today; fields arrive with the round that renders them | reserved |
| `encodeAcknowledged` | the person has seen once that boxes / overlays force a re-encode | desktop |

## 10. Who reads what

| Reader | Reads | Ignores |
|---|---|---|
| Desktop (`apps/desktop/shared/project/project-document.ts migrateProject`) | everything; upgrades 2 / 3 → 4 | — |
| Rust exporter (`crates/cutsign-backend/src/model.rs`, `export.rs`) | `assets`, `sourceOrder`, `segments`, `presentation.redactions` / `notes` / `overlayClips` / `audio` | `preset`, `groups`, `editor`, `preprocessingRuns`, `silenceThreshold`, provenance |
| Engine (`src/video_preprocessor/desktop/api.py preprocess_project`) | `assets`, `segments` | `preset`, `presentation` |
| CLI writer (`services/audit_project.py`) | writes v4 from an analysis run (`--pre`, `--preset`) | — |

The `required` lists in the schema are the union of what every reader needs: a document that validates opens in the
desktop and exports in Rust (cross references aside, see §11).

## 11. Validating a file

The schema is a test-time arbiter; the app does not validate at runtime. To check a document you wrote:

```bash
# Node (ajv 8, strict mode, no formats)
npx ajv-cli validate -s contracts/project-file/project.v4.schema.json -d my.cutsign.json --spec=draft2020 --strict=true
# Python
python -c "import json,jsonschema;jsonschema.validate(json.load(open('my.cutsign.json')),json.load(open('contracts/project-file/project.v4.schema.json')))"
```

The schema cannot express cross references. Check yourself that every `segments[].assetId`, `sourceOrder[]` entry and
`groups[].assetIds[]` entry names an `assets[].id` (`fixtures/invalid/orphan-segment.json` validates but is broken). A
`cutsign validate` command that does both is planned (BRD `OPEN_EDITOR_FOR_CREATORS_AND_AGENTS` P3).

## 12. Change rules

1. A shape change bumps `version`, adds a fixture and turns the three language test suites red then green. An optional field
   every reader can ignore does not bump, but enters the schema in the same commit as the code that writes it.
2. One coded error for an unknown version, identical in the desktop, Rust and the engine (§2.3).
3. Every enum lists every value any writer emits.
4. No `format` keywords; timestamps are checked by a loose ISO-8601 pattern.
5. `fixtures/cli-audit.v4.json` is generated (`scripts/generate_contract_fixtures.py`), never hand-edited; its diff is the
   contract change.
