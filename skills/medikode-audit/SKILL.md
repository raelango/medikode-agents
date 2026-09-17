---
name: medikode-audit
description: Run the Medikode audit/reconciliation pipeline (S11 Recon Normalizer through S15 Recon Narrative Packager) to check AI-generated codes against human-coded codes and chart evidence, flagging unsupported or mismatched codes before a claim goes out. Takes the same inputs as the demo app's "Audit Codes with Medical Chart" form and produces the same per-stage outputs. Stage prompts and schemas are read live from the raelango/medikode-agents GitHub repo, and the run is recorded to that same repo instead of SharePoint. Use when the user invokes /medikode-audit, asks to "audit" or "reconcile" a coded chart, or asks to check AI codes against a human coder's codes.
---

# medikode-audit

Reconciles two sets of finalized claim lines for the same encounter — one
produced by the AI coding pipeline, one by a human coder — against the
original chart, and adjudicates every discrepancy: which side is more
likely correct, why they disagree, and what to do about it. This is the
"Code Auditor" capability in the Medikode product ("Verifies assigned
codes against chart documentation to catch unsupported codes before
claims go out").

Like `medikode-code`, you act as the model for every stage yourself,
chaining each stage's JSON output into the next. In the real app, "audit"
mode runs the full S1-S10 coding pipeline exactly as `medikode-code` does
(the AI codes the chart with zero knowledge of the human's answer), and
only then runs S11-S15 to compare the two. Do the same, in the same order.

## Step 1 — Load stage definitions

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use the same shared cache dir as the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, and always `pull` first.

Read every `audit/S1*.json` file (S11-S15), keep `enabled: true`, sort by
`sequence`. Each file has the same shape as `medikode-code`'s stage files
(`stage_id`, `title`, `slug`, `sequence`, `enabled`, `system_prompt`,
`user_prompt`, `output_schema`, `input_keys`, `variable_map`,
`max_output_tokens`, `temperature`, `notes`).

## Step 2 — Gather inputs

This is the same form as `medikode-code` (specialty, encounter type,
patient status, site of care, claim type, facility, insurance,
insurance type, chart text) **plus** the human coder's codes. If the user
already has S1/S2/S3/S10 outputs from a `medikode-code` run earlier in this
conversation for the same chart, reuse them; otherwise run `medikode-code`'s
Step 1-4 yourself first (same inputs) to produce them — tell the user
you're doing this. That includes `medikode-code`'s own specialty/insurance
guideline lookup against `reference/specialties.json`/`insurances.json` and
its facility-type sanity check — nothing extra to do here for those.

Additionally, ask the user directly for:

- `human_coded_input` — the human coder's final codes, as a multiline
  string, one code per line (formats like `99214-25-E11.69,I10` or
  `* G0446-59-I10`).

## Step 3 — Build the two claim inputs exactly as the real app does

Don't hand S11 the raw upstream packages or the raw human text directly —
the real app pre-shapes both into a common `{results: [...]}` form first:

- `ai_final_claim` = `{"results": S10.final_claim_package.final_claim_lines + S10.final_claim_package.final_quality_lines}`
  (concatenate the two arrays from your `medikode-code` run's S10 output)
- `human_final_claim` = `{"results": [...], "human_submission_context": {}}`,
  where each line of `human_coded_input` becomes one result object using
  this exact deterministic parsing (matches the real app's
  `buildHumanClaimLines`):
  - `cpt_code` = the raw line string (trim leading `*`/whitespace)
  - `coding_system` = `CPT_II` if it matches `^[0-9]{4}[A-Z]$`, else
    `HCPCS` if it starts with a letter (e.g. G, J, A), else `CPT_I`
  - `line_type` = `QUALITY` if `coding_system` is `CPT_II`, else `BILLABLE`
  - `units` = the number after `*` in the line, default `1`
  - `modifier` = the value after `-` in the line, default `""`

Also carry forward from the `medikode-code` run: `chart_canonical_package`
(S1), `claim_context_package` (S2), `coder_facts_package` (S3) — S13/S14
need these for evidence.

Optional inputs used by later stages (default to empty/unknown if not
given, per each stage's own `variable_map`/options defaults documented in
its `notes`):

- `diff_options`, `evidence_binding_options`, `adjudication_options`,
  `packaging_options`, `scoring_policy` — each stage's JSON file documents
  its own expected shape and sensible defaults in `user_prompt`/`notes`;
  use those defaults unless the user specifies otherwise.
- `policy_context` — payer/specialty profile ids and ruleset versions
  (from the `medikode-code` run's S6/S7/S8 outputs, if available;
  otherwise leave fields empty as the real product does today per S14's
  `notes`).

## Step 4 — Run each stage in sequence

Same mechanics as `medikode-code`: for each stage in `sequence` order,
render `user_prompt` by resolving every `{{variable}}` via `variable_map`
against the running `inputs` bag, adopt `system_prompt`, and produce only
the JSON `output_schema` asks for — grounded in the real chart, real codes,
and prior stage outputs, not placeholders. Store each stage's single
top-level output key both back into the `inputs` bag and into a
`stage_results` dict keyed by stage id, mirroring `medikode-code`'s Step 4.

S11 and S12 are meant to be strictly deterministic (parsing/diffing with
no judgment calls) — follow their parsing rules exactly rather than
improvising. S13-S15 involve more judgment (evidence binding,
adjudication, narrative) — stay evidence-bound: only cite excerpt IDs that
actually exist in `chart_canonical_package`, never invent codes, and use
only the enum values each stage's schema specifies for taxonomy fields
(disposition, root_cause_tag, recommended_action, etc.).

The 5 stages, in order: Recon Normalizer (S11) → Recon Differ (S12) →
Recon Evidence Binder (S13) → Recon Adjudicator (S14) → Recon Narrative
Packager (S15).

## Step 5 — Record the run (GitHub datastore)

Per `DATASTORE.md` in the repo: write a `submissions/audit/<yyyy>/<mm>/<id>.json`
record with `request: {variables: {...the Step 2 fields..., human_coded_input}, content: chart_text, use_cache}`
and `response: stage_results` for S11-S15 (the coding pipeline's own S1-S10
run should already have been recorded under `submissions/code/...` by
`medikode-code` itself). Append one line to `audit/log.jsonl`. Commit and
push.

## Step 6 — Report results

Present each stage's package (S11 through S15), matching how the real app
shows tabs per stage rather than one condensed answer:

- Overall match rate and how many lines were AI-only, human-only, or unit
  mismatches (from S12/S14)
- The adjudicated items: which side was likely correct, root cause, and
  recommended action for each meaningful discrepancy (from S14)
- The top issues and any provider-query packet from S15's narrative
  package (`recon_report_package`), and its denial-risk framing

## Notes

- To change pipeline behavior, edit `audit/S1*.json` in
  `raelango/medikode-agents` — not this skill file.
- `human_final_claim` is treated as "post-validation" per S11's prompt —
  don't second-guess it as wrong unless deterministic policy/math or clear
  chart contra-evidence makes that unambiguous (S14's rule).
