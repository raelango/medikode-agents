---
name: medikode-validate
description: Check a set of already-coded CPT/HCPCS/ICD-10 code combinations for compliance issues, checking NCCI/PTP bundling against a real ~3.1M-row NCCI PTP dataset (not just model knowledge) plus MUE/modifier/diagnosis-specificity checks from general knowledge. Takes the same input as the demo app's "Validate Code Combinations" form and produces the same response shape (valid_codes/invalid_codes/etc). Uses the V1 stage prompt and validate/data/ read live from the raelango/medikode-agents GitHub repo. This skill only reads from that repo — it never writes or pushes anything to it. Use when the user invokes /medikode-validate, asks to validate code combinations/pairs, or asks for a compliance check on a set of billing codes.
---

# medikode-validate

Checks a set of already-assigned codes for compliance before claim
submission — the kind of check a payer's claims-editing engine runs
(NCCI/PTP bundling edits, MUE unit limits, missing required modifiers,
add-on codes without their primary, unspecified diagnoses). This is the
"Validation Agent" in the Medikode product ("Checks code-pair combinations
against compliance edits and payer-specific rules in real time").

**Important caveat, surface this to the user before running:** the real
product resolves "validate" mode to an internal configuration record
(its actual instructions live in a `context_prompt`/`output_prompt` pair
on that record, calling out to an external agent) — there is no source
prompt for it anywhere in this repo to migrate faithfully. The
`validate/V1-*.json` stage definition's *system/user prompt* in the
GitHub repo is therefore a from-scratch reconstruction based on the
product's own description of what it does and standard NCCI/CCI/MUE
claims-editing practice. Its *output shape*, however, is not a guess —
it's grounded in the real app's rendering code, which reads exactly
`valid_codes`, `invalid_codes`, `summary`, `coding_year`,
`denial_risk`, `final_codes`, `code_conflicts`, `issues`, and
`recommended_changes` off the reply.

As of `schema_version` 1.2.0, the **PTP-bundling check specifically is no
longer just the model's general knowledge** — Step 3 below looks up every
pair of billed procedure codes against a real, ~3.1M-row NCCI PTP dataset
(`validate/data/`) and hands confirmed matches to the model as ground
truth. MUE, missing-modifier, add-on-code, and diagnosis-specificity
checks are still best-effort general knowledge, since no equivalent
ground-truth dataset exists for those yet. So: treat PTP findings as
high-confidence, everything else as a reasonable best-effort check, and
the *output shape* as a faithful match to the real app either way.

This is a single-stage pipeline — there's no chaining, but it still
follows the "definition lives in the GitHub repo, not this skill file"
pattern as the other medikode-* skills.

## Step 1 — Load the stage definition

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use the same shared cache dir as the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, and always `pull` first. This
repo uses Git LFS for the large files under `validate/data/` — a normal
`git clone`/`git pull` fetches LFS content automatically when git-lfs is
installed on the machine; if `validate/data/ncci_ptp_*.txt` looks like a
tiny pointer stub (a few lines of text, not megabytes of pipe-delimited
data) rather than real data, run `git lfs pull` in the cache dir and
check `git lfs install` has been run at least once on the machine.

Read `validate/V1-validate-codes.json`. It has `system_prompt`,
`user_prompt` (with `{{coded_input}}` and `{{ptp_edits_found}}`
placeholders), `output_schema`, and `variable_map`.

## Step 2 — Gather inputs

Ask the user for `coded_input`: the set of coded lines to validate, one
per line — formats like `99214-25-E11.69,I10,E78.5,D64.9` (CPT-MODIFIER-
DX1,DX2,...) or `* G0446-59-I10`. This is the only field the real app's
"Validate Code Combinations" form collects — no chart text, no
specialty/insurance/facility context.

## Step 3 — Look up real PTP edits

Extract the procedure code from every line of `coded_input`: split the
line on `-` and take the first segment (trimmed of whitespace/leading
`*`) — that's the CPT/HCPCS code (the format is
`CPT-MODIFIER-DX1,DX2,...`, so later `-`-separated segments are
modifiers/diagnoses, not procedure codes). Collect the **unique** set of
procedure codes billed.

If there are fewer than 2 unique procedure codes, there's nothing to
pair-check — set `ptp_edits_found` to an empty array and skip to Step 4.

Otherwise, for every unique unordered pair `(A, B)` among them, check
both directions against both files (a pair may only be listed one way
round in the source data):

```
grep -m1 "^A|B|" <cache-dir>/validate/data/ncci_ptp_practitioner.txt
grep -m1 "^B|A|" <cache-dir>/validate/data/ncci_ptp_practitioner.txt
grep -m1 "^A|B|" <cache-dir>/validate/data/ncci_ptp_hospital.txt
grep -m1 "^B|A|" <cache-dir>/validate/data/ncci_ptp_hospital.txt
```

(substitute the real codes for `A`/`B` — plain `grep`, no `-E`/`-F`
needed, since these files have no regex-special characters other than the
literal `|` delimiters, which default `grep` treats literally.) For every
match, parse `column1|column2|modifierIndicator|rationaleCode`, resolve
`rationaleCode` against `validate/data/rationale_codes.json`, and append
`{code_a: column1, code_b: column2, modifier_indicator, rationale,
code_set: "Practitioner"|"OutpatientHospital"}` to `ptp_edits_found`. A
pair can legitimately produce both a practitioner and a hospital match
(or neither) — include every match found, don't dedupe across code sets.

## Step 4 — Run the stage

Render `user_prompt` with `coded_input` and `ptp_edits_found` (as a JSON
array, `[]` if none found) substituted in, adopt `system_prompt`, and
produce only the JSON `output_schema` asks for: split every input line
into either `valid_codes` or `invalid_codes` (never both), and for
anything invalid also add a `code_conflicts`/`issues` entry explaining
why and a corrected line (or its omission) in `final_codes` so
`final_codes` always reflects your recommended, compliant code set. Every
entry in `ptp_edits_found` must show up as a `PTP_EDIT` finding — don't
let the model second-guess a confirmed real-data match. Also fill
`summary`, `coding_year` (if inferable, else leave blank), and
`denial_risk` (Low/Medium/High). Beyond the confirmed PTP edits, only
flag other issues you're reasonably confident about — this is a
compliance pre-check, not a full coding review.

## Step 5 — Report results

Give the user the valid vs. invalid split, ordered by severity, plus the
recommended `final_codes` set and overall `denial_risk`. Call out which
`PTP_EDIT` findings came from the confirmed real-data lookup (Step 3) vs.
any other issue types that are the model's general knowledge — that
distinction matters for how much to trust each finding. If nothing was
found invalid, say so plainly rather than padding the response with
low-confidence findings.

## Notes

- To change this stage's prompt, edit `validate/V1-validate-codes.json` in
  `raelango/medikode-agents` — not this skill file. Given the caveat
  above, the *prompt* (not the output shape) is a good candidate to
  refine further if the user has access to the real assistant's actual
  context_prompt/output_prompt.
- `validate/data/` covers PTP edits only — there's no equivalent
  ground-truth dataset yet for MUE unit limits, add-on-code-requires-
  primary rules, or modifier-required rules. Those checks stay
  general-knowledge-based until/unless such data gets added the same way.
- This skill only ever reads from `raelango/medikode-agents` — it never
  commits, pushes, or otherwise writes to it.
