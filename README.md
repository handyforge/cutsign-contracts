# CutSign contracts

Public data contracts of [CutSign](https://handyforge.app/cutsign), the desktop video editor by handyforge: the
JSON Schema of the project file, the fixtures every reader is tested against, and a specification written for people
and agents who have never seen the app.

| Path | What |
|---|---|
| `project-file/SPEC.md` | **Start here.** What a `.cutsign.json` is, the stability promise, every field, who reads what, a minimal valid document, how to validate one |
| `project-file/project.v4.schema.json` | JSON Schema 2020-12 of project versions 2, 3 and 4 (`$id` `https://handyforge.app/cutsign/contracts/project-file/project.v4.schema.json`) |
| `project-file/project.v3.schema.json` | Previous schema, kept for history |
| `project-file/analysis.v1.schema.json` | The engine's analysis payload (version 1) |
| `project-file/fixtures/` | Positive fixtures (`desktop.v4.json` carries every optional field once; `cli-audit.v4.json` is generated), `legacy.v2.json`, `asset-id.json` (pins the asset id hash), `invalid/` negatives |
| `export-pieces/fixtures/pieces.v1.json` | Test vectors for how kept segments become export pieces (merge, frame snap, fps) |

## Stability promise

The schema major version changes only with the app major version. Within a major, no required field is added, no field
is removed or retyped, no enum value is removed; optional fields may be added. The app reads every version it ever wrote
and refuses a newer one with one coded error (`[project.version_unsupported] …`). Details and dates in `SPEC.md` §2.

## Change rules

1. A shape change bumps `version`, adds a fixture and turns the app's three language test suites red then green. An
   optional field every reader can ignore does not bump, but enters the schema in the same change as the code that writes it.
2. Every enum lists every value any writer emits.
3. No `format` keywords; timestamps are checked by a loose ISO-8601 pattern.
4. `fixtures/cli-audit.v4.json` is generated, never hand-edited; its diff is the contract change.

## Validate a file

```bash
npx ajv-cli validate -s project-file/project.v4.schema.json -d my.cutsign.json --spec=draft2020 --strict=true
```

Fixtures contain the placeholder `{{root}}` at the start of every media path; replace it with a folder of your own.

## Source and licence

This repository mirrors `contracts/` of the CutSign application repository; changes land there first and are copied
here with each release. Schemas and documentation are MIT (`LICENSE`); fixtures are CC0 (`project-file/fixtures/LICENSE`).
The application itself is proprietary and is not in this repository.

## MCP: drive the running app

| Path | What |
|---|---|
| `mcp/MCP.md` | How to connect Claude Code, Codex, Cursor or Gemini CLI to a running CutSign (`CutSign.exe --mcp`), the security model, error codes, and the tool reference generated from the app's tool schemas |
| `mcp/cutsign-skill/SKILL.md` | The agent skill: the read → edit → look → person-reviews loop, every tool with its parameters, the workspace rules, rendering overlays from a brief |

Both files mirror `docs/MCP.md` and `docs/agent/cutsign-skill/SKILL.md` of the application repository.
