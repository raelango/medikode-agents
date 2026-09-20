# validate/data/

Real NCCI Procedure-to-Procedure (PTP) edit pairs backing `medikode-validate`'s
PTP-bundling check. Stored via Git LFS (see the repo's `.gitattributes`) since
these are ~22-28MB each — a normal `git clone`/`git pull` fetches the LFS
content automatically when git-lfs is installed; run `git lfs pull` by hand
if a file looks like a small pointer stub instead of real data.

## Files

- `ncci_ptp_practitioner.txt` — 1,732,834 edit pairs, professional/practitioner
  code set.
- `ncci_ptp_hospital.txt` — 1,406,713 edit pairs, outpatient hospital code set.
- `rationale_codes.json` — the 12 distinct rationale categories referenced by
  both files above, keyed by the short numeric code used in column 4.

## Format

Each line of the two `.txt` files is one edit pair, pipe-delimited, sorted by
`column1Code` then `column2Code`:

```
column1Code|column2Code|modifierIndicator|rationaleCode
```

- `column1Code` / `column2Code` — the CPT/HCPCS code pair. `column2Code` is
  the one that's normally bundled into `column1Code` when both are billed for
  the same patient, same date of service, same provider.
- `modifierIndicator` — `0` (edit cannot be bypassed with a modifier — billing
  both codes together is essentially never appropriate) or `1` (edit can be
  bypassed with an appropriate modifier, e.g. 59/XE/XP/XS/XU, when clinically
  justified).
- `rationaleCode` — an integer key into `rationale_codes.json` explaining why
  the edit exists (e.g. "Mutually exclusive procedures", "More extensive
  procedure").

A pair might only appear in one direction (`A|B|...`) even though the two
codes could be billed in either order — always check both `A|B` and `B|A`
when looking up a code pair a user submitted.

## Provenance

One-time export, filtered from a larger internal dataset (3,139,547 total
source rows across both files) down to currently-active edits: any row whose
deletion date was a real date in the past (not `*`) was dropped — none were
found expired as of this export (2026-09-19), so this run's filter had no
effect, but it's applied on every rebuild as a safety net rather than trusting
the source's own "active" pre-filtering alone. Only the fields
`medikode-validate` actually needs were kept (source rows also carried
`priorTo1996`, `effectiveDate`, `status`, `source`, and a `codeSet` label —
dropped as redundant/unused here). Rebuilding this export requires access to
the original internal dataset, which is not itself checked into this repo.

## Using this data in a skill

See `skills/medikode-validate/SKILL.md` — it greps both files for every pair
of billed codes in the user's input before running the V1 stage, and feeds
any real matches into the prompt as ground truth.
