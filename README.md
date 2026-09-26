# Medikode Agents

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**Stage definitions and Claude Code skills for Medikode's agentic medical-coding pipelines.**

This repository is the source of truth for the AI agents behind [Medikode.ai](https://medikode.ai). It defines each pipeline stage (prompts, output schemas, sequencing, and model settings) and ships those pipelines as installable Claude Code skills. Coding, audit, remittance, risk adjustment, and compliance checks each run as multi-step agent workflows, with human coders in the loop for complex cases.

---

## Pipelines

| Folder | Stages | Pipeline | What it does |
|---|---|---|---|
| `coding/` | S1 – S10 | **Chart-to-claim** | Reads unstructured clinical notes and produces validated ICD-10 / CPT codes ready for the claim |
| `audit/` | S11 – S15 | **Reconciliation** | Compares AI-assigned codes with human coding and explains the differences |
| `era/` | E1 – E7 | **Remittance** | Converts free-text EOBs / remittances into EDI 835 |
| `raf/` | R1 | **Risk adjustment** | Calculates Risk Adjustment Factor (RAF) scores |
| `validate/` | V1 | **Code-pair compliance** | Checks code pairs against CMS NCCI PTP edits |

Supporting folders:

- `skills/` — Claude Code skills: `medikode-code`, `medikode-audit`, `medikode-era`, `medikode-raf`, `medikode-validate`
- `reference/` — reference data (specialties, payers, facilities) used by the coding pipeline
- `validate/data/` — about 3.1 million CMS NCCI PTP edit pairs, practitioner and hospital (Git LFS), so compliance checks use real CMS rules, not model recall

## How a stage is defined

Every stage is a declarative file that the pipeline runner executes in order:

- System and user prompts
- Output schema, so every stage returns structured, validated JSON
- Sequencing and dependencies on earlier stages
- Model settings, including token limits and temperature

Because the stages are data rather than code, prompts and schemas can be versioned, reviewed, and improved without redeploying the platform.

## Install the skills

**Claude Code plugin marketplace (recommended, updates automatically):**

```
/plugin marketplace add raelango/medikode-agents
```

**Manual:** copy the folders under `skills/` into `~/.claude/skills/`.

## Design principles

- **No PHI in version control.** The repo never stores patient data or clinical content. An earlier write-back feature was removed so chart excerpts and AI-generated codes can't be committed by accident.
- **Read-only skills.** Skills read stage definitions from this repo and never write, commit, or push to it.
- **Grounded, not guessed.** Compliance checks run against real CMS NCCI data rather than relying on the model's memory.
- **Human in the loop.** Low-confidence and complex cases go to human coders instead of being auto-finalized.
- **Structured outputs.** Schema-validated JSON at every stage makes results auditable and easy to integrate downstream.

## Related

- [medikode-mcp-server](https://github.com/raelango/medikode-mcp-server) — exposes Medikode's coding tools to Claude, Cursor, and ChatGPT over the Model Context Protocol

## License

MIT © Medikode.ai
