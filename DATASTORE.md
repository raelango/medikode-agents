# Datastore conventions

The Medikode web app persists three things to SharePoint on every pipeline
run: a **submission** record (request + response), an **audit log** entry,
and (optionally) a **cache** entry keyed by the request so identical
requests can short-circuit. The `medikode-*` Claude Code skills replace all
three with plain files in this repo, committed and pushed by the skill
itself at the end of each run — no SharePoint/Graph API dependency.

This file is the shared spec all five `medikode-*` skills follow. Each
skill's own instructions just say "record per DATASTORE.md" rather than
repeating this.

## submissions/

One file per run: `submissions/<mode>/<yyyy>/<mm>/<id>.json`, where `<mode>`
is `code`, `audit`, `era`, `raf`, or `validate`, and `<id>` is
`sub_<unix-timestamp>_<6-char-random>` (e.g. `sub_1758100000_a1b2c3`).

```json
{
  "id": "sub_1758100000_a1b2c3",
  "mode": "code",
  "created_at": "2026-09-17T14:32:00Z",
  "actor_email": "user@example.com",
  "status": "success",
  "request": {
    "variables": { "...": "mode-specific fields, see each skill's Step 2" },
    "content": "the raw chart/coded-input/EOB text submitted",
    "use_cache": true
  },
  "response": {
    "...": "mode-specific — see each skill's Step 4 for the exact shape"
  },
  "error": null
}
```

- `actor_email`: best-effort — use the git user's email
  (`git config user.email`) if available, else omit the field.
- On failure, set `status: "error"`, put the failure reason in `error`,
  and set `response` to `null`.
- This mirrors SharePoint's `create_submission` + `update_submission` as a
  single record (a skill run is one interactive turn, not a
  server handling concurrent requests, so there's no separate
  pending/complete phase to preserve).

## cache/

Before running a pipeline, compute
`cache_key = sha256(json.dumps({mode, variables, content}, sort_keys=true))`
and check whether `cache/<cache_key>.json` exists in the repo. If it does
and the user hasn't asked to bypass the cache (mirrors the UI's "Use Cache"
checkbox, default on), read its `response` field and skip straight to
reporting results instead of re-running the pipeline — tell the user
you're using a cached result.

```json
{
  "cache_key": "<sha256 hex>",
  "mode": "raf",
  "created_at": "2026-09-17T14:32:00Z",
  "response": { "...": "same shape as the submission's response" }
}
```

After a successful run (not a cache hit), write this file so future
identical requests can reuse it.

## audit/log.jsonl

Append one line (a compact single-line JSON object, no pretty-printing) per
run to `audit/log.jsonl` — this is a JSON Lines file, append-only, so
concurrent runs from different machines merge as normal git line-additions
rather than conflicting on a shared JSON array/object.

```json
{"timestamp": "2026-09-17T14:32:00Z", "action": "medikode.<mode>.run", "mode": "code", "status": "success", "actor_email": "user@example.com", "submission_id": "sub_1758100000_a1b2c3", "detail": "short one-line summary, e.g. final_claim_package.finalized=true"}
```

## Committing

After writing the relevant `submissions/`, `cache/`, and `audit/log.jsonl`
files, `git add` them, commit with a short message like
`medikode-<mode>: record run <id>`, and push. Use the same shared clone as
the stage-definition pull (`~/.claude/skills/.medikode-agents-cache`) —
`pull --ff-only` immediately before committing to minimize the chance of a
non-fast-forward push if another run happened concurrently; if the push is
rejected, pull again and retry once, and otherwise tell the user the
record couldn't be pushed rather than silently dropping it.
