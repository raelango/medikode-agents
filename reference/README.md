# reference/

Reference/lookup data the coding pipeline's form fields draw on in the real
app (populating dropdowns, and the specialty/payer guideline text used by
`coding/S6`/`S7`). Pulled one-time, live, from the SharePoint lists the
backend (`backend/app/sharepoint_client.py`) reads today, via the same
Graph API credentials.

| File | Source SharePoint list | Shape |
|---|---|---|
| `specialties.json` | Specialties | Array of `{id, title, guidelines, encounter_types[], initial_codes_system_prompt}`. `guidelines` is the specialty-specific coding ruleset text (this is what `coding/S6`'s `guidelines` variable wants). |
| `insurances.json` | Insurances | Array of `{id, title, payer_bundling_policy, specimen_collection_policy, qw_requirement_policy, vaccine_funding_source_policy, payer_types[], primary_payer_type, vaccine_funding_apply_to, guidelines, created, modified}`. `guidelines` feeds `coding/S7`. |
| `facilities.json` | Facilities | Array of `{facility_type, encounter_types[], site_of_care[], default_claim_type, specialties[]}` — the facility-type taxonomy (valid encounter types / site of care / claim type per facility type, and which specialties apply to it). |
| `vaccine_components.json` | Vaccine Components | Array of `{item_id, title, cpt, component_count, description}` — multi-component vaccine CPT codes and how many billable components each bundles (used by the 90460/90461 admin-code bundling rule in `backend/app/validations.py`). |

## Not migrated

**S1 Header Mappings** was deliberately left out — several of its records
contain what looks like real chart-excerpt text (patient allergies,
medications, procedures tied to a specific encounter) rather than the
generic section-header strings ("Subjective", "History of Present
Illness", ...) the list is meant to cache for header-detection reuse. That
looks like a data-hygiene issue in the source SharePoint list itself and
should be fixed there before any export — not something to publish to a
git repo.

## Using this data in a skill

None of the `medikode-*` skills read `reference/` automatically yet — as
of this migration, `medikode-code`/`medikode-audit` still ask the user to
type in specialty/insurance guideline text directly (mirroring the real
app's behavior of fetching it once a dropdown is picked, just without the
dropdown). Wiring a skill to look guidelines up from here by name instead
of asking the user to paste them is a natural follow-up, not yet done.
