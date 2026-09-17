---
name: medikode-validate
description: Check a set of already-coded CPT/HCPCS/ICD-10 code combinations for compliance issues (NCCI/PTP bundling, MUE unit limits, missing modifiers, unspecified diagnoses), taking the same input as the demo app's "Validate Code Combinations" form and producing the same response shape (valid_codes/invalid_codes/etc). Uses the V1 stage prompt read live from the raelango/medikode-agents GitHub repo, and records the run to that same repo instead of SharePoint. Use when the user invokes /medikode-validate, asks to validate code combinations/pairs, or asks for a compliance check on a set of billing codes.
---

# medikode-validate

Checks a set of already-assigned codes for compliance before claim
submission — the kind of check a payer's claims-editing engine runs
(NCCI/PTP bundling edits, MUE unit limits, missing required modifiers,
add-on codes without their primary, unspecified diagnoses). This is the
"Validation Agent" in the Medikode product ("Checks code-pair combinations
against compliance edits and payer-specific rules in real time").

**Important caveat, surface this to the user before running:** the real
product resolves "validate" mode to an assistant record configured
entirely in SharePoint (its actual instructions live in a `context_prompt`/
`output_prompt` pair on that record, calling out to an external agent) —
there is no source prompt for it anywhere in the codebase to migrate
faithfully. The `validate/V1-*.json` stage definition's *system/user
prompt* in the GitHub repo is therefore a from-scratch reconstruction
based on the product's own description of what it does and standard
NCCI/CCI/MUE claims-editing practice. Its *output shape*, however, is not
a guess — it's grounded in the real frontend rendering code, which reads
exactly `valid_codes`, `invalid_codes`, `summary`, `coding_year`,
`denial_risk`, `final_codes`, `code_conflicts`, `issues`, and
`recommended_changes` off the reply. So treat the prompt's *reasoning* as
a reasonable best-effort check, but its *output shape* as a faithful match
to the real app.

This is a single-stage pipeline — there's no chaining, but it still
follows the "definition lives in the GitHub repo, not this skill file"
pattern as the other medikode-* skills.

## Step 1 — Load the stage definition

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use the same shared cache dir as the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, and always `pull` first.

Read `validate/V1-validate-codes.json`. It has `system_prompt`,
`user_prompt` (with a `{{coded_input}}` placeholder), `output_schema`, and
`variable_map`.

## Step 2 — Gather inputs

Ask the user for `coded_input`: the set of coded lines to validate, one
per line — formats like `99214-25-E11.69,I10,E78.5,D64.9` (CPT-MODIFIER-
DX1,DX2,...) or `* G0446-59-I10`. This is the only field the real app's
"Validate Code Combinations" form collects — no chart text, no
specialty/insurance/facility context. Also ask about `use_cache`
(default `true`).

## Step 3 — Check the cache, then run the stage

Compute `cache_key = sha256(json.dumps({mode:"validate", variables: {}, content: coded_input}, sort_keys=true))`
(the real app sends an empty `variables` object for this mode — all
context is in `content`). Per `DATASTORE.md`: if `use_cache` is true and
`cache/<cache_key>.json` exists, use its `response` and skip to Step 5.

Otherwise render `user_prompt` with `coded_input` substituted in, adopt
`system_prompt`, and produce only the JSON `output_schema` asks for: split
every input line into either `valid_codes` or `invalid_codes` (never
both), and for anything invalid also add a `code_conflicts`/`issues` entry
explaining why and a corrected line (or its omission) in `final_codes` so
`final_codes` always reflects your recommended, compliant code set. Also
fill `summary`, `coding_year` (if inferable, else leave blank), and
`denial_risk` (Low/Medium/High). Only flag issues you're reasonably
confident about — this is a compliance pre-check, not a full coding
review.

## Step 4 — Record the run (GitHub datastore)

Per `DATASTORE.md`: write `submissions/validate/<yyyy>/<mm>/<id>.json`
with `request: {variables: {}, content: coded_input, use_cache}` and
`response` = the stage's JSON output. If not a cache hit, also write
`cache/<cache_key>.json`. Append a line to `audit/log.jsonl`. Commit and
push.

## Step 5 — Report results

Give the user the valid vs. invalid split, ordered by severity, plus the
recommended `final_codes` set and overall `denial_risk`. If nothing was
found invalid, say so plainly rather than padding the response with
low-confidence findings.

## Notes

- To change this stage's prompt, edit `validate/V1-validate-codes.json` in
  `raelango/medikode-agents` — not this skill file. Given the caveat
  above, the *prompt* (not the output shape) is a good candidate to
  refine further if the user has access to the real assistant's actual
  context_prompt/output_prompt.
