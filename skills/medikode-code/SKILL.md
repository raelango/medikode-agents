---
name: medikode-code
description: Run the Medikode multi-stage medical coding pipeline (S1 Prep Chart through S10 Finalize Claim) against a patient chart, taking the same inputs as the Medikode demo app's "Code Medical Records" form and producing the same per-stage outputs. Stage prompts, schemas, and sequencing are read live from the raelango/medikode-agents GitHub repo instead of the old SharePoint "Coding Pipeline Stages" list, and the run is recorded to that same repo instead of SharePoint. Use when the user invokes /medikode-code, asks to run the Medikode coding pipeline, or asks to "code this chart" end to end.
---

# medikode-code

Runs a chart through the Medikode coding pipeline: a fixed sequence of
stages, each with its own system prompt, user prompt template, and JSON
output schema. You act as the "model" for every stage yourself, chaining
each stage's JSON output into the inputs of the next stage — this mirrors
exactly how the product's own demo app runs it client-side (its `code`/
`audit` modes call each stage directly, stage by stage, rather than one
big backend call).

The stage definitions are **not** hardcoded in this skill — they live in
the `raelango/medikode-agents` GitHub repo (`coding/*.json`) so they can be
edited without touching this skill file. Always fetch the current versions
before running; don't rely on a stale local copy. That repo is also where
this skill records each run (see Step 5) — there's no SharePoint/Graph API
dependency anywhere in this skill.

This repo also holds stage definitions for four sibling pipelines (`audit/`,
`era/`, `raf/`, `validate/`), each with its own skill (`medikode-audit`,
`medikode-era`, `medikode-raf`, `medikode-validate`). This skill only reads
`coding/`.

## Step 1 — Load stage definitions and reference data

Get a fresh copy of the repo:

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use a stable cache dir shared with the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, so repeated runs across any of
the medikode-* skills are fast, but always `pull` first so edits made in
the repo are picked up. If git/network access fails and a cached copy
exists, warn the user you're using a possibly-stale cache and continue; if
no cache exists, stop and tell the user you can't reach the repo.

Read every `coding/S*.json` file, parse it, keep only `enabled: true`
stages, and sort ascending by `sequence`. Each stage file has:

- `stage_id`, `title`, `slug`, `sequence`, `enabled`
- `system_prompt`, `user_prompt` (template with `{{variable}}` placeholders)
- `output_schema` (the JSON shape the stage must return)
- `input_keys` (names of values the stage consumes)
- `variable_map` (maps each `{{variable}}` in `user_prompt` to an `inputs.<key>`
  path — that's where its value comes from)
- `max_output_tokens`, `temperature`, `notes`, `schema_version`

Also read `reference/specialties.json`, `reference/insurances.json`, and
`reference/facilities.json` — this is a one-time export of the same
SharePoint lists the real app's dropdowns/guideline lookups draw on (see
`reference/README.md`). Step 2 uses these to resolve guideline text
automatically instead of asking the user to paste it.

## Step 2 — Gather inputs

These mirror exactly the fields on the demo app's "Code Medical Records"
form (verified against the app's own client-side pipeline code, not
guessed):

**Required:**
- `chart_text` — the raw chart/encounter note (ask for it, or read it if
  the user gave a file path). This is the only input S1 uses.

**Optional** (ask once up front; default any not given to `""` — later
stages are written to treat an empty value as "unknown", same as the real
app does today for every version/id field below):

- `specialty` — clinical department/specialty for the encounter
- `encounter_type` — type of visit
- `patient_status` — New or Established
- `site_of_care` — place of service
- `claim_type` — claim category
- `facility` — a small object describing the facility:
  `{name, facility_type, facility_teaching_status, locations: [], providers: [], guidelines}`.
  `guidelines` (facility-specific free text) is used directly; the rest
  feed `source_metadata` below.
- `insurance` — payer name
- `insurance_type` — plan type (e.g. HMO/PPO)
- `use_cache` — boolean, default `true` (see Step 5 — mirrors the app's
  "Use Cache" checkbox)

Don't block the run over missing optional inputs.

**Resolve guidelines from `reference/` instead of asking for them** — this
is what the real app does too (it fetches these once a dropdown value is
picked, rather than making the user paste guideline text):

- `specialty_guidelines`: find the entry in `reference/specialties.json`
  whose `title` matches `specialty` case-insensitively. If found, use its
  `guidelines` field verbatim. If not found (or `specialty` was left
  blank), tell the user no on-file guidelines matched and ask if they want
  to paste guideline text manually or proceed without any — don't guess a
  close match silently.
- `insurance_guidelines`: find the entry in `reference/insurances.json`
  whose `title` matches `insurance` case-insensitively. If found, render
  it as a short text block (not just the raw `guidelines` field, since
  the payer's policy fields matter here too):
  ```
  Insurance: {title}
  Payer Type(s): {payer_types joined by ", ", or "Unknown"}
  Primary Payer Type: {primary_payer_type or "Unknown"}
  Specimen Collection Policy: {specimen_collection_policy}
  QW Requirement Policy: {qw_requirement_policy}
  Vaccine Funding Source Policy: {vaccine_funding_source_policy}
  Vaccine Funding Applies To: {vaccine_funding_apply_to}
  Guidelines:
  {guidelines, or "(none on file)"}
  ```
  If no entry matches, same fallback as above: tell the user, offer manual
  paste or proceeding without.

If the user explicitly provides `specialty_guidelines`/`insurance_guidelines`
text themselves, that overrides the lookup — don't discard what they gave
you in favor of a reference-data match.

**Sanity-check the facility context** — if `facility.facility_type` is set,
look it up in `reference/facilities.json`. If found, and any of
`encounter_type`, `site_of_care`, `claim_type`, or `specialty` isn't in
that facility type's `encounter_types`/`site_of_care`/`default_claim_type`/
`specialties` lists, mention the mismatch to the user before running
(it's a real signal something was mistyped or misselected) — but don't
block the run over it, since the pipeline can still execute.

## Step 3 — Build the running inputs bag and check the cache

Compute `cache_key = sha256(json.dumps({mode:"code", variables, content: chart_text}, sort_keys=true))`
where `variables` is every optional field from Step 2. Per `DATASTORE.md`
in the repo: if `use_cache` is true and `cache/<cache_key>.json` exists,
read its `response`, skip straight to Step 6 with that cached per-stage
output, and tell the user you used a cached result.

Otherwise, build the running `inputs` bag stage by stage exactly as the
real app does:

- `source_metadata` (used by S1 and S2) = `{facility, encounter_type,
  patient_status, site_of_care, claim_type, specialty, insurance,
  insurance_type, guidelines: facility.guidelines}`
- S1: `raw_chart_text` = `chart_text`, `source_metadata` as above
- S2: `chart_canonical_package` (S1's output), `source_metadata`
- S3: `chart_canonical_package` (S1), `claim_context_package` (S2),
  `coder_facts_schema_version` = `"1.0.0"` (fixed)
- S4: `coder_facts_package` (S3), `claim_context_package` (S2)
- S5: `coder_facts_package` (S3), `claim_context_package` (S2),
  `specialty_name` = `specialty`, `guidelines` = every string field of
  `facility` flattened into one text blob (name, type, teaching status,
  etc. — the real app calls this `extractStringFields`), plus
  `code_crosswalk_bundle_id`, `code_crosswalk_bundle_version`,
  `baseline_ruleset_version` = `""` (the real app sends these empty today
  — this isn't a gap in your run, it's current behavior)
- S6: `baseline_code_candidates` (S5), `specialty_name` = `specialty`,
  `guidelines` = `specialty_guidelines`, `specialty_profile_version` = `""`
- S7: `specialty_adjusted_codes` (S6), `guidelines` = 
  `"Insurance Type : {insurance_type}\nGuidelines:\n{insurance_guidelines}"`,
  `payer_profile_id` = `insurance`, `payer_profile_version` = `""`,
  `payer_profile_effective_date` = `""`, `edits_engine_version` = `""`
- S8: `payer_adjusted_codes` (S7), `chart_canonical_package` (S1),
  `coder_facts_package` (S3), `verification_ruleset_version` = `""`
- S9: `coding_readiness_report` (S4), `payer_adjusted_codes` (S7),
  `verification_outcome` (S8), `coder_facts_package` (S3),
  `chart_canonical_package` (S1), `scoring_policy_version` = `""`
- S10: `payer_adjusted_codes` (S7), `verification_outcome` (S8),
  `case_disposition_package` (S9), `coding_readiness_report` (S4)

(The real app also threads a `previous_response_id` between stages for its
own backend's response-caching — that's an implementation detail of its
OpenAI Responses API usage, not something you need to replicate.)

## Step 4 — Run each stage in sequence

For each stage, in `sequence` order:

1. Render `user_prompt`: for every `{{variable}}` it contains, resolve it
   from the `inputs` bag built in Step 3 (JSON-stringify objects/arrays,
   use scalars as-is).
2. Adopt the stage's `system_prompt` as your instructions for this step.
3. Produce **only** the raw JSON object the prompt asks for — no markdown
   fences, no commentary — matching `output_schema`'s shape, filled in with
   real content derived from the chart and prior stage outputs (not
   placeholder text). Treat `temperature: 0` as "be deterministic and
   literal."
4. Parse your own JSON output (one top-level key, e.g.
   `chart_canonical_package`) and store it in the `inputs` bag under that
   name, and also into a `stage_results` dict keyed by stage id (e.g.
   `stage_results["S1"] = {...chart_canonical_package}`) — this
   `stage_results` dict is your final output shape, matching the real
   app's own `stageResults[stage.id].artifact` structure exactly.
5. If a stage's output signals a hard failure (e.g. S4's
   `coding_readiness_report.status == "fail"`, or S8's
   `verification_outcome.status == "fail"` with `hard_failures`), keep
   going through the remaining stages (they're designed to carry
   review/failure state forward to S9/S10) but flag this clearly in your
   final summary.

The 10 stages, in order: Prep Chart (S1) → Billing Context (S2) → Extract
Facts (S3) → Check Completeness (S4) → Initial Codes (S5) → Specialty
Rules (S6) → Payer Edits (S7) → Verify Codes (S8) → Notes Queries (S9) →
Finalize Claim (S10).

## Step 5 — Record the run (GitHub datastore)

Per `DATASTORE.md` in the repo: write a `submissions/code/<yyyy>/<mm>/<id>.json`
record with `request: {variables, content: chart_text, use_cache}` and
`response: stage_results` (the full per-stage dict from Step 4). If this
wasn't a cache hit, also write `cache/<cache_key>.json` with
`response: stage_results`. Append one line to `audit/log.jsonl`. Commit and
push. This replaces the SharePoint "Submissions"/"Audit Logs" writes the
real app makes on every run.

## Step 6 — Report results

The real app has no single condensed "final answer" for this pipeline —
like it, present each stage's package (the tabs it would show), not just a
narrative:

- Show (or offer to show in full) each of `stage_results.S1` through
  `.S10`
- Highlight `S10.final_claim_package.finalized` and the final billable /
  quality lines
- `S9.case_disposition_package`'s confidence/denial-risk scores and
  routing recommendation (finalize vs. review)
- Any hard failures, missing inputs, or review triggers surfaced along
  the way (from S4, S8, S9), and which stage raised them

## Notes

- To change pipeline behavior (prompts, schemas, sequencing, enabling/
  disabling a stage), edit the corresponding file in
  `raelango/medikode-agents` (`coding/S*.json`) — not this skill file —
  and it takes effect on the next run.
- The `audit` pipeline (`medikode-audit` skill) builds on this one: it
  needs S1/S2/S3/S10's outputs from a `medikode-code` run as part of its
  own inputs. In the real app, the human coder's codes (`codedInput`)
  never enter S1-S10 at all — audit mode runs this exact same pipeline
  blind to the human's answer, and only compares against it afterward at
  S11. Do the same: don't let a human code list influence this run.
- `reference/vaccine_components.json` also exists in the repo but isn't
  used by this skill: it backs a 90460/90461 multi-component-vaccine
  bundling correction (`backend/app/validations.py`'s `_validate_90461`)
  that the real app only ever applies to its older, non-v2 "code" response
  shape — the v2 pipeline this skill replays never actually calls it
  today, so wiring it in here would add behavior the live pipeline doesn't
  have, not match it.
- To change which reference data is available, edit
  `raelango/medikode-agents/reference/*.json` — see `reference/README.md`
  for how each file was sourced and its refresh status (these are a
  one-time export, not a live sync).
