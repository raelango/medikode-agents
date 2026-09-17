---
name: medikode-era
description: Run the Medikode remittance/EOB pipeline (E1 Text Ingestor through E7 Lossless 835 Packager) to turn free-text EOB or remittance content into a structured, evidence-traced package and a real ANSI X12 835 file. Takes the same input as the demo app's "Generate ERA" form. Stage prompts and schemas are read live from the raelango/medikode-agents GitHub repo, and the run is recorded to that same repo instead of SharePoint. Use when the user invokes /medikode-era, asks to "generate an ERA", parse an EOB/remittance, or turn remittance text into an 835.
---

# medikode-era

Turns free-text EOB (Explanation of Benefits) / remittance content into a
structured, evidence-addressable package, ending in a real ANSI X12 835
file. This is the "Remittance Agent" in the Medikode product ("Converts
EOB documents into standardized ERA output").

Like `medikode-code`, you act as the model for every stage yourself,
chaining each stage's JSON output into the next.

**Important caveat, surface this to the user before running:** the real
product's live "era" mode is a *single* opaque call to an externally
configured assistant (its instructions live entirely in a SharePoint
record, not in this codebase), whose reply then goes through a legacy
JS function to produce EDI text — and that function has a real bug: it
computes a nicely-normalized document shape (`sender`/`recipient`/
`summary`/`claim_information`) but then never actually uses it, reading a
completely different, undocumented flat shape (`payerInfo`/`payeeInfo`/
`checkInfo`/`claims[]`) instead, with several non-standard field reuses
(e.g. its SVC04 holds an allowed amount, not the revenue code X12 defines
there). The `era/E1-E7` pipeline in the GitHub repo is a more rigorous,
evidence-traced *replacement* design (documented in this repo's engineering
docs) that was never actually wired into the live app. This skill runs
that more rigorous pipeline and produces genuinely correct X12 835 output
from it — it does not attempt to reproduce the legacy function's bug.
If exact byte-parity with today's live (flawed) output is specifically
wanted, see the note at the end of Step 5 instead.

## Step 1 — Load stage definitions

```
git -C <cache-dir> pull --ff-only  ||  git clone --depth 1 https://github.com/raelango/medikode-agents.git <cache-dir>
```

Use the same shared cache dir as the sibling skills, e.g.
`~/.claude/skills/.medikode-agents-cache`, and always `pull` first.

Read every `era/E*.json` file, keep `enabled: true`, sort by `sequence`.
Same file shape as the other pipelines (`stage_id`, `title`, `slug`,
`sequence`, `enabled`, `system_prompt`, `user_prompt`, `output_schema`,
`input_keys`, `variable_map`, `max_output_tokens`, `temperature`, `notes`).

## Step 2 — Gather inputs

These mirror the demo app's "Generate ERA" form:

- `remit_text` (required) — the raw EOB/remittance text (ask for it, or
  read it if the user gave a file path). This is the app's `content`.
- `facility` — a small object, same shape as in `medikode-code`:
  `{name, facility_type, facility_teaching_status, locations: [], providers: [], guidelines}`
- `use_cache` — boolean, default `true`

## Step 3 — Check the cache

Build `variables = {client: facility.name, facility: facility.name,
facility_type: facility.facility_type, facility_teaching_status:
facility.facility_teaching_status, facility_locations: facility.locations,
facility_providers: facility.providers, guidelines: facility.guidelines}`
(matches the real app's field names exactly) and compute
`cache_key = sha256(json.dumps({mode:"era", variables, content: remit_text}, sort_keys=true))`.
Per `DATASTORE.md` in the repo: if `use_cache` is true and
`cache/<cache_key>.json` exists, use its `response` and skip to Step 6.

## Step 4 — Run each stage in sequence

Same mechanics as `medikode-code`. `metadata` for E1 = `{source_channel:
"ui", facility: facility.name}` (matches real wiring per E1's `notes`); a
few things specific to this pipeline, called out in individual stages'
`notes` — read each stage's `notes` before running it, but in particular:

- **E1 (Text Ingestor)**: its schema wants a real `sha256_raw_text` hash
  and `char_length`/`line_count` integrity fields. Don't have the model
  invent a hash — after E1 produces its JSON, compute the actual SHA-256
  of `remit_text` yourself (e.g. via a shell command) and overwrite the
  `integrity.sha256_raw_text` / `idempotency.dedupe_key` fields with the
  real value before storing this stage's output.
- **E2, E5, E6**: their source docs describe these as deterministic,
  no-LLM stages. You're standing in for that deterministic logic — follow
  each stage's stated rules exactly and literally rather than treating
  them as creative/open-ended. E5 in particular builds a complete,
  correctly-structured X12 835 AST (envelope + segments) — this is what
  Step 5 renders into real EDI text, so get its segment arrays right.
- **E3**: only allowed to use LLM-style judgment as a bounded fallback in
  the original design; treat its no-fabrication / must-cite-spans
  constraints as hard requirements.
- **E7**: its schema also wants real content hashes over the assembled
  package. As with E1, compute these for real after the model produces
  the rest of the structure, rather than trusting a model-generated hash.

Store each stage's single top-level output key into the running `inputs`
bag, and into a `stage_results` dict keyed by stage id, exactly as in
`medikode-code`.

The 7 stages, in order: Text Ingestor (E1) → Text Structurer (E2) → Remit
Canonicalizer (E3) → Remit Context Assembler (E4) → X12 Model Builder (E5)
→ Financial Integrity Validator (E6) → Lossless 835 Packager (E7).

## Step 5 — Render the real EDI-835 file

E5's `x12_model_package.x12` already contains a complete, correctly
modeled 835 transaction as segment arrays plus the delimiters to use
(`x12.profile.delimiters`, typically `*` for elements and `~` for
segments). Serialize it directly — this is mechanical, not another model
call:

1. For each segment (the envelope's `ISA`, `GS`, `GE`, `IEA` arrays, and
   per transaction: `ST`, each entry in `segments[]` — use its `elements`
   array — then `SE`), join that segment's elements with the element
   delimiter.
2. Join all segments in order (ISA, GS, then each transaction's ST +
   segments + SE, then GE, IEA) with the segment delimiter, and terminate
   the file with a trailing segment delimiter.

This produces genuine ANSI X12 835 text, ready to hand to the user or pipe
into another EDI tool — it's the actual deliverable "era" mode exists to
produce, not just a JSON description of one.

**If exact parity with today's live (flawed) output is specifically
wanted instead:** that would mean building a flat `{payerInfo: {id, name,
qualifier}, payeeInfo: {npi, taxId, name}, checkInfo: {checkNumber,
checkDate, checkAmount, paymentMethod, routingNumber, accountNumber},
claims: [{patientControlNumber, claimStatus, chargeAmount, paidAmount,
patientResponsibility, patientInfo: {...}, adjustments: [...],
serviceLines: [{procedureCode, chargeAmount, paidAmount, allowedAmount,
units, dateOfService, adjustments: [...]}]}]}` document instead, and
running it through the same fixed default-filling/hardcoded-field rules
the legacy generator uses (e.g. CLP06 always `"MC"`, CLP08 always `"11"`,
CLP09 always `"1"`, missing payer id defaults to the literal string
`"PAYERID"`, dates must be ISO `YYYY-MM-DD` or they silently fall back to
today's date). Only go this route if the user explicitly asks to match
today's live output rather than get a correct 835.

## Step 6 — Record the run (GitHub datastore)

Per `DATASTORE.md`: write `submissions/era/<yyyy>/<mm>/<id>.json` with
`request: {variables, content: remit_text, use_cache}` and `response` =
`stage_results` (include the rendered EDI-835 text alongside it, e.g. as
`response.edi835_text`). If not a cache hit, also write
`cache/<cache_key>.json`. Append a line to `audit/log.jsonl`. Commit and
push.

## Step 7 — Report results

- Whether E6's financial integrity check passed (totals/balancing) and
  any findings it raised
- The key remittance facts assembled along the way (payer/payee, claims,
  adjustments, amounts) from E3/E4
- The rendered EDI-835 text from Step 5, and offer to show any
  intermediate stage package in full on request
- Any `warnings`/`errors`/`missing_inputs` surfaced by any stage

## Notes

- To change pipeline behavior, edit `era/E*.json` in
  `raelango/medikode-agents` — not this skill file.
- Several stages in this pipeline were originally deterministic Python
  services, not LLM calls; each stage's `notes` field documents that and
  any known fidelity gap from now running them as an LLM instead (hashing
  in particular — see Step 4).
