# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Changes to `skills/prd-to-automation/references/contract.md` are listed under a **Contract**
heading and bump the contract version stated in that file.

## [Unreleased]

## [1.0.0] - 2026-09-22

First release of the **prd-to-automation** skill: one Claude Code skill with the modes govern,
automate, and audit. It merges two earlier skills with **unchanged behaviour**:

| Origin | Version | Now |
|---|---|---|
| [`prd-to-governance`](https://github.com/Karlderkarl/prd-to-governance) | 1.2.0 | govern mode |
| [`governance-to-automation`](https://github.com/Karlderkarl/governance-to-automation) | 1.2.2 | automate mode |
| the audit parts of both | | audit mode |

Projects whose governance was generated with `prd-to-governance` 1.2.0 and whose script was
generated with `governance-to-automation` 1.2.2 keep working without change; an audit reports no
drift merely because of the skill change.

### Added
- `skills/prd-to-automation/SKILL.md`: short entry point with the project-root rule, the mode
  table, the mode-selection rules, the boundaries, and the references each mode loads. Frontmatter
  uses only `name`, `description`, `license`, `metadata.version`.
- `references/audit.md`: read-only mode that combines the governance drift report, the script drift
  report (six drift classes plus contract gaps and safety invariants), and validation (`bash -n`,
  `shellcheck`, `--dry-run`) into one report with proposed mode switches.
- `agents/openai.yaml` for the new name (display name, short description, default prompt).
- `docs/PRD.md` (the PRD this repository implements, German) and `docs/parity.md` (every rule of
  both origin quality checklists and every Critical invariant, mapped to its new location).
- `skills.sh.json` with the group "Governance"; MIT `LICENSE`; `SECURITY.md` with GitHub Private
  Vulnerability Reporting.

### Contract
- `references/contract.md`, **contract version 1.0.0**. Defines once what govern writes, automate
  reads, and audit checks: the five uncertainty markers, the three priority levels, the memory rules
  (M1 to M7), the AGENTS.md *Skill Policy* format and how it seeds `SKILL_MAP`, the test-discipline
  fields (`TEST_POLICY`, `TEST_ELIGIBILITY`, `TARGETED_TEST_CMD` with `{TARGET}`), and a field table
  stating for every field whether it is optional or required and what happens when it is absent or
  partial. The content is exactly what the origins produced and consumed; consolidating it is not a
  behaviour change.

### Moved (origin part → new location)
- `prd-to-governance` `SKILL.md` header, file table, generation order, Steps 0 to 8 and 10 to 11,
  adaptation guidelines, quality checklist → `references/govern.md`. Its "Modes" section became
  govern's sub-modes Generate and Update/merge.
- `prd-to-governance` `SKILL.md` Step 9 (audit process, drift categories, audit targets, contract
  coherence checks) → `references/audit.md` Part A. `govern.md` Step 9 links to it.
- `prd-to-governance` `SKILL.md` "Uncertainty Markers" and "Priority Levels" →
  `references/contract.md` sections 1 and 2.
- `prd-to-governance` `references/soul-template.md`, `agents-template.md`, `claude-template.md`,
  `memory-template.md`, `completed-phases-template.md` → `references/` (verbatim; two sentences in
  `agents-template.md` and one in `claude-template.md` now say "the automate mode" instead of
  naming the other skill).
- `governance-to-automation` `SKILL.md` intro, artifact table, "Governance is the contract",
  "Deterministic skill resolution", "Two-pass pipeline", Steps 0 to 7, adaptation guidelines,
  quality checklist → `references/automate.md`. Its "Modes" Generate and Audit/Sync became
  automate's sub-modes Generate and Sync; its Validate mode became audit Part C.
- `governance-to-automation` `SKILL.md` "Memory discipline is the core integration" (the rules) →
  `references/contract.md` section 3; `automate.md` keeps the Critical statement and links to it.
- `governance-to-automation` `SKILL.md` "Uncertainty and priority markers" → `contract.md`
  sections 1 and 2.
- `governance-to-automation` `SKILL.md` "Audit/Sync mode" (six drift classes, contract gaps) →
  `references/audit.md` Part B; `automate.md` Sync links to it.
- `governance-to-automation` `references/auto-develop-template.md`, `prompt-builders.md`,
  `task-list-template.md`, `extraction-checklist.md` → `references/` (verbatim; "SKILL.md Step N"
  pointers now read "`automate.md` Step N", "route back to prd-to-governance" now reads "switch
  to govern mode", and the two pointers to the origin's *Memory discipline* and *Deterministic skill
  resolution* sections now point to `contract.md` sections 3 and 4).
- `governance-to-automation` `examples/auto-develop.payload-sample.sh`, `examples/refact-todo.md`
  → `examples/` (verbatim except the header comment, which now names the automate mode and the
  new blueprint path).
- `governance-to-automation` `agents/openai.yaml` → `agents/openai.yaml` (renamed fields).
- `governance-to-automation` root `CLAUDE.md`, `AGENTS.md`, `SECURITY.md`, `skills.sh.json` were
  the templates for this repository's files of the same name.

### Changed
- Repo and skill are named `PRD-to-automation` / `prd-to-automation`, decided on 2026-09-22 (PRD section 6).
  The PRD proposed `governance-pipeline`; that name stays with the pi sister project
  `pi-governance-pipeline`.
- Every cross-skill reference ("route back to `prd-to-governance`", "recommend
  `governance-to-automation`", "separate skill") is now a mode switch ("switch to the govern mode",
  "switch to the automate mode"). The origin names appear in the skill only as provenance.
- The skill frontmatter carries `license: MIT` and `metadata.version` (the
  `governance-to-automation` frontmatter had neither).

[1.0.0]: https://github.com/Karlderkarl/PRD-to-automation/releases/tag/v1.0.0
