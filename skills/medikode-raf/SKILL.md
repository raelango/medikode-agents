---
name: medikode-raf
description: Compute a CMS-HCC Risk Adjustment Factor (RAF) score from patient demographics and diagnoses, taking the same inputs as the demo app's "Calculate RAF Score" form and producing the same response shape. Uses the R1 stage prompt read live from the raelango/medikode-agents GitHub repo, and records the run to that same repo instead of SharePoint. Use when the user invokes /medikode-raf, asks for a RAF score, or asks about CMS-HCC risk adjustment for a patient.
---

# medikode-raf

Computes a CMS-HCC Risk Adjustment Factor (RAF) score: maps each ICD-10-CM
diagnosis to its HCC category under the requested model version, applies
demographic and HCC coefficients plus interaction terms, and sums to a
final score. This is the "Risk Adjustment Agent" in the Medikode product.

This is a single-stage pipeline — there's no chaining, but it still
follows the same "definition lives in the GitHub repo, not this skill
file" pattern as the other medikode-* skills.

## Step 1 — Load the stage definition

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use the same shared cache dir as the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, and always `pull` first.

Read `raf/R1-raf-score.json`. It has `system_prompt`, `user_prompt` (with
`{{demographics}}`, `{{illnesses}}`, `{{model}}` placeholders),
`output_schema`, and `variable_map`.

## Step 2 — Gather inputs

These are exactly the three fields on the demo app's "Calculate RAF Score"
form:

- `demographics` — age and gender at minimum
- `illnesses` — the ICD-10-CM diagnoses to score
- `model` — which CMS-HCC model version: `V28` or `V24`. Default to `V28`
  if the user doesn't specify. Reject/ask again on anything else — these
  are the only two supported model versions.
- `use_cache` — boolean, default `true` (see Step 3)

## Step 3 — Check the cache, then run the stage

Compute `cache_key = sha256(json.dumps({mode:"raf", variables: {demographics, illnesses, model}, content: "demographics : {demographics}\nIllnesses : {illnesses}\nModel: {model}"}, sort_keys=true))`
— this content string matches exactly what the real app sends. Per
`DATASTORE.md` in the repo: if `use_cache` is true and
`cache/<cache_key>.json` exists, use its `response` and skip to Step 5,
telling the user you used a cached result.

Otherwise render `user_prompt` by substituting the three variables, adopt
`system_prompt`, and produce only the JSON `output_schema` asks for:
demographic segment + coefficient, each HCC mapping with its coefficient,
any interaction terms, a step-by-step `calculation` narrative, and the
final summed RAF score. Round all coefficient/score values to 4 decimal
places, per the prompt. This JSON (`{raf, model_used, calculation,
hcc_categories}`) is exactly the reply shape the real app's backend
returns for this mode — no reshaping needed.

## Step 4 — Record the run (GitHub datastore)

Per `DATASTORE.md`: write `submissions/raf/<yyyy>/<mm>/<id>.json` with
`request: {variables: {demographics, illnesses, model}, content, use_cache}`
and `response` = the stage's JSON output. If not a cache hit, also write
`cache/<cache_key>.json`. Append a line to `audit/log.jsonl`. Commit and
push.

## Step 5 — Report results

Give the user the final RAF score, the demographic/HCC/interaction
breakdown, and flag any diagnosis codes that had no HCC mapping (these
don't contribute to the score but are worth surfacing, since it can mean
either the code genuinely doesn't risk-adjust or documentation is missing
a more specific code).

## Notes

- To change this stage's prompt, edit `raf/R1-raf-score.json` in
  `raelango/medikode-agents` — not this skill file.
- This was migrated verbatim from the backend's `/rafscore` endpoint
  prompt — high fidelity to the real product behavior, and its output
  shape matches what the demo app's own backend returns for raf mode
  exactly.
