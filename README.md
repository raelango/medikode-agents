# medikode-agents

Metadata store **and skill source** for the Medikode pipeline agents. This
repo is the source of truth for both the **stage definitions** (system/user
prompts, output schemas, sequencing) and the **Claude Code skill
definitions** themselves for each Medikode pipeline, replacing an internal
configuration list as the config store.

Each skill pulls its stage definitions (and, where relevant, `reference/`
data) from here at the start of every run and runs the pipeline itself,
acting as each stage's model in turn. **These skills only ever read from
this repo — none of them write, commit, or push anything to it.** (An
earlier version had each skill record its run here as a GitHub-backed
datastore; that was removed — a run's request/response can include real
patient chart text, which has no business being committed into a git
repo.)

## Installing as a Claude Code plugin

This repo is also a Claude Code plugin marketplace — the fastest way to
install these skills:

```
/plugin marketplace add raelango/medikode-agents
/plugin install medikode-code@medikode-agents
/plugin install medikode-audit@medikode-agents
/plugin install medikode-era@medikode-agents
/plugin install medikode-raf@medikode-agents
/plugin install medikode-validate@medikode-agents
```

Each skill is its own plugin so you can install only the ones you need.
See `.claude-plugin/marketplace.json` for the marketplace definition and
each `skills/<name>/.claude-plugin/plugin.json` for the individual plugin
manifests. Installing this way keeps itself up to date the normal plugin
way; manually copying a skill folder into `~/.claude/skills/` (below)
still works too, it just won't auto-update.

## Layout

```
coding/     S1  - S10   Code a chart end to end (Prep Chart -> Finalize Claim)
audit/      S11 - S15   Reconcile AI-coded vs. human-coded claims against chart evidence
era/        E1  - E7    Turn free-text EOB/remittance content into a lossless 835 package
raf/        R1          Compute a CMS-HCC Risk Adjustment Factor score
validate/   V1          Check code-pair combinations for compliance
skills/     medikode-code, medikode-audit, medikode-era, medikode-raf,
            medikode-validate — the canonical SKILL.md for each Claude Code
            skill, each also wrapped as its own plugin (see above). Manual
            install/update: copy the folder into ~/.claude/skills/.
reference/  Specialties, insurances, facilities, vaccine components — the
            reference lists the coding pipeline reads today. See
            reference/README.md.
validate/data/  Real NCCI PTP edit-pair data (Git LFS) backing
            medikode-validate's PTP check. See validate/data/README.md.
.claude-plugin/  Marketplace manifest (marketplace.json) for the plugin
            install method above.
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

## Updating a skill

Edit `skills/<name>/SKILL.md` here, then copy it over the installed copy
at `~/.claude/skills/<name>/SKILL.md` (Claude Code loads skills from disk
at session start, so this repo copy alone doesn't take effect until it's
copied in).

## History

- 2026-09-11: `coding/` (S1-S10) migrated one-time from an internal
  configuration list.
- 2026-09-11: `audit/` (S11-S15) and `era/` (E1-E7) extracted from
  internal engineering specs and cross-checked against the real client
  orchestration wiring. `raf/` (R1) migrated verbatim from the backend's
  RAF-scoring endpoint prompt. `validate/` (V1) has no known source
  prompt anywhere accessible (the product's "Validate Code Combinations"
  mode calls an external, opaque service) — it was written from scratch
  based on the product's own description of what it does and standard
  NCCI/CCI/MUE claims-editing practice, and should be treated as a draft
  to refine rather than a faithful migration.
- 2026-09-17: skills redesigned to take the same inputs as the demo app's
  own forms and produce the same output shapes (verified against the
  app's actual client-side code and rendering logic, not guessed), and
  added `skills/` (the SKILL.md source for each). `validate/V1`'s output
  schema was corrected to match the real `valid_codes`/`invalid_codes`/...
  contract (an earlier draft used non-matching field names). Also
  discovered the live "era" mode's actual EDI generator reads a
  different, undocumented field shape than its own normalization code
  produces (a real bug in the product, not this repo) — see
  `skills/medikode-era/SKILL.md` for details; the `era/E1-E7` pipeline
  here is a correct alternative, not a bug-for-bug replica.
  (This same change briefly added a `DATASTORE.md` + `submissions/`/
  `cache/`/`audit/` write-back convention; it was removed the same day —
  see below.)
- 2026-09-17: `reference/` added — one-time export of Specialty,
  Insurance, Facility, and Vaccine Component reference data from an
  internal system, the data the "code"/"audit" pipelines'
  specialty/insurance/facility fields draw on in the real app. One
  related internal list was deliberately NOT migrated: several of its
  records contain what looks like real chart-excerpt text (allergies,
  medications, procedures) rather than the generic section-header
  strings it's meant to hold — a data-hygiene issue worth fixing at the
  source, not something to publish here.
- 2026-09-17: removed the GitHub-write-back feature (`DATASTORE.md` and
  the "record the run" step in every skill) added earlier the same day.
  A `code`/`audit` submission record would include the raw patient chart
  text and AI-generated diagnosis/procedure codes — real clinical content
  that has no business being committed into a git repo, regardless of the
  repo's visibility. No run had actually completed under that design, so
  nothing needed to be purged from history. All five skills are read-only
  against this repo now.
- 2026-09-19: internal-system references scrubbed from the working tree
  (this repo is public) — replaced with generic phrasing throughout. Git
  history was left as-is (not rewritten).
- 2026-09-19: `validate/data/` added — a real NCCI Procedure-to-Procedure
  (PTP) edits dataset (Git LFS), trimmed from ~868MB of source data down
  to ~48MB (1,732,834 practitioner + 1,406,713 hospital edit pairs).
  `medikode-validate`'s PTP-bundling check is no longer just the model's
  general knowledge — it now greps this data for every pair of billed
  codes and feeds confirmed matches into the V1 prompt as ground truth
  (`validate/V1-validate-codes.json` bumped to schema_version 1.2.0).
  MUE/modifier/add-on checks remain general-knowledge-based.
- 2026-09-20: `.claude-plugin/marketplace.json` added, plus a
  `.claude-plugin/plugin.json` in each `skills/<name>/` directory, so this
  repo doubles as a Claude Code plugin marketplace (`/plugin marketplace
  add raelango/medikode-agents`). Each skill is registered as its own
  plugin rather than one bundled plugin, so users can install only what
  they need.
