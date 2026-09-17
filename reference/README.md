# reference/

Reference/lookup data the coding pipeline's form fields draw on in the real
app (populating dropdowns, and the specialty/payer guideline text used by
`coding/S6`/`S7`). Pulled one-time from an internal system.

| File | Shape |
|---|---|
| `specialties.json` | Array of `{id, title, guidelines, encounter_types[], initial_codes_system_prompt}`. `guidelines` is the specialty-specific coding ruleset text (this is what `coding/S6`'s `guidelines` variable wants). |
| `insurances.json` | Array of `{id, title, payer_bundling_policy, specimen_collection_policy, qw_requirement_policy, vaccine_funding_source_policy, payer_types[], primary_payer_type, vaccine_funding_apply_to, guidelines, created, modified}`. `guidelines` feeds `coding/S7`. |
| `facilities.json` | Array of `{facility_type, encounter_types[], site_of_care[], default_claim_type, specialties[]}` — the facility-type taxonomy (valid encounter types / site of care / claim type per facility type, and which specialties apply to it). |
| `vaccine_components.json` | Array of `{item_id, title, cpt, component_count, description}` — multi-component vaccine CPT codes and how many billable components each bundles (used by a 90460/90461 admin-code bundling rule in the backend). |

## Not migrated

One related internal reference list was deliberately left out — several of
its records contain what looks like real chart-excerpt text (patient
allergies, medications, procedures tied to a specific encounter) rather
than the generic section-header strings ("Subjective", "History of Present
Illness", ...) it's meant to cache for header-detection reuse. That looks
like a data-hygiene issue in the source list itself and should be fixed
there before any export — not something to publish to a git repo.

## Using this data in a skill

`medikode-code`/`medikode-audit` resolve `specialty_guidelines` and
`insurance_guidelines` by looking up the entered specialty/insurance name
in `specialties.json`/`insurances.json`, instead of asking the user to
paste guideline text — see those skills' `SKILL.md` for the exact lookup
and rendering rules. They also cross-check `encounter_type`/`site_of_care`/
`claim_type`/`specialty` against `facilities.json`. `vaccine_components.json`
isn't wired into any skill yet (see `skills/medikode-code/SKILL.md`'s
Notes for why).
