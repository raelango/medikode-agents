# medikode-agents

Metadata store for the Medikode coding pipeline agents. This repo is the
source of truth for the pipeline **stage definitions** (system/user prompts,
output schemas, sequencing) that used to live only in a SharePoint list
(`Coding Pipeline Stages`). It's read by the `medikode-code` Claude Code
skill (installed as a personal skill) to run the pipeline without needing
SharePoint/Graph API access.

## Layout

```
stages/
  S1-prep-chart.json
  S2-billing-context.json
  S3-extract-facts.json
  S4-check-completeness.json
  S5-initial-codes.json
  S6-specialty-rules.json
  S7-payer-edits.json
  S8-verify-codes.json
  S9-notes-queries.json
  S10-finalize-claim.json
```

Each `stages/S*.json` file describes one pipeline stage:

| Field | Meaning |
|---|---|
| `stage_id` | Short id (e.g. `S1`) |
| `title` | Human-readable stage name |
| `slug` | URL/file-safe slug |
| `sequence` | Execution order (ascending) |
| `enabled` | Whether the stage is active |
| `schema_version` | Version of the stage's output schema |
| `prompt_format` | Format of the model output (`json`) |
| `system_prompt` | System prompt for the stage |
| `user_prompt` | User prompt template, with `{{variable}}` placeholders |
| `output_schema` | Expected JSON output shape |
| `max_output_tokens` | Output token cap |
| `temperature` | Model temperature |
| `notes` | Free-form notes |
| `input_keys` | Names of inputs the stage consumes |
| `variable_map` | Maps each `{{variable}}` in `user_prompt` to where its value comes from |

## Updating stage definitions

Edit the relevant `stages/S*.json` file and commit. There is no separate
index file — stage order and enabled state are read directly from each
file's `sequence` / `enabled` fields.

## History

The stage definitions were migrated one-time from the SharePoint
`Coding Pipeline Stages` list on 2026-09-11.
