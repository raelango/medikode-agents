# medikode-agents

Metadata store for the Medikode pipeline agents. This repo is the source of
truth for **stage definitions** (system/user prompts, output schemas,
sequencing) for each Medikode pipeline. It's read by the `medikode-code`,
`medikode-audit`, `medikode-validate`, `medikode-raf`, and `medikode-era`
Claude Code skills (installed as personal skills), each of which pulls the
current stage definitions from here and runs the pipeline itself, acting as
each stage's model in turn.

## Layout

```
coding/     S1  - S10   Code a chart end to end (Prep Chart -> Finalize Claim)
audit/      S11 - S15   Reconcile AI-coded vs. human-coded claims against chart evidence
era/        E1  - E7    Turn free-text EOB/remittance content into a lossless 835 package
raf/        R1          Compute a CMS-HCC Risk Adjustment Factor score
validate/   V1          Check code-pair combinations for compliance
```

Each pipeline's directory holds one JSON file per stage, named
`<stage_id>-<slug>.json`. A stage file looks like:

| Field | Meaning |
|---|---|
| `stage_id` | Short id (e.g. `S1`, `E3`, `R1`) |
| `title` | Human-readable stage name |
| `slug` | URL/file-safe slug |
| `sequence` | Execution order within its pipeline (ascending) |
| `enabled` | Whether the stage is active |
| `schema_version` | Version of the stage's output schema |
| `prompt_format` | Format of the model output (`json`) |
| `system_prompt` | System prompt for the stage |
| `user_prompt` | User prompt template, with `{{variable}}` placeholders |
| `output_schema` | Expected JSON output shape |
| `max_output_tokens` | Output token cap |
| `temperature` | Model temperature |
| `notes` | Free-form notes, including any known fidelity gaps vs. the original implementation (e.g. a stage that used to run deterministic Python and is now LLM-driven) |
| `input_keys` | Names of inputs the stage consumes |
| `variable_map` | Maps each `{{variable}}` in `user_prompt` to where its value comes from |

There's no separate index file per pipeline — stage order and enabled state
are read directly from each file's `sequence` / `enabled` fields.

## Pipeline dependencies

- `audit/` (S11-S15) consumes outputs from `coding/` (S1's
  `chart_canonical_package`, S2's `claim_context_package`, S3's
  `coder_facts_package`, S10's `final_claim_package` as `ai_final_claim`)
  plus a user-supplied `human_final_claim` that has no upstream stage.
- `coding/`, `era/`, `raf/`, and `validate/` are each self-contained.

## Updating stage definitions

Edit the relevant `<pipeline>/<stage_id>-<slug>.json` file and commit.

## History

- 2026-09-11: `coding/` (S1-S10) migrated one-time from the SharePoint
  `Coding Pipeline Stages` list.
- 2026-09-11: `audit/` (S11-S15) and `era/` (E1-E7) extracted from this
  repo's own engineering handoff docs (`Artifacts/S11-*.md` .. `S15-*.md`,
  `Artifacts/E1-*.md` .. `E7-*.md`) and cross-checked against the real
  client orchestration wiring. `raf/` (R1) migrated verbatim from the
  backend's `/rafscore` endpoint prompt. `validate/` (V1) has no known
  source prompt anywhere in the codebase (the product's "Validate Code
  Combinations" mode calls an external, opaque service) — it was written
  from scratch based on the product's own description of what it does and
  standard NCCI/CCI/MUE claims-editing practice, and should be treated as a
  draft to refine rather than a faithful migration.
