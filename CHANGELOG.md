# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Changes to `skills/prd-to-automation/references/contract.md` are listed under a **Contract**
heading and bump the contract version stated in that file.

## [Unreleased]

## [1.0.0] - 2026-09-22

First release of the **prd-to-automation** skill: one Claude Code skill with the modes govern,
automate, and audit. It merges two earlier skills with **unchanged behaviour**, except for the
deliberate fixes listed under *Fixed*:

| Origin | Version | Now |
|---|---|---|
| [`prd-to-governance`](https://github.com/Karlderkarl/prd-to-governance) | 1.2.0 | govern mode |
| [`governance-to-automation`](https://github.com/Karlderkarl/governance-to-automation) | 1.2.2 | automate mode |
| the audit parts of both | | audit mode |

Projects whose governance was generated with `prd-to-governance` 1.2.0 and whose script was
generated with `governance-to-automation` 1.2.2 keep working without change; an audit reports no
drift merely because of the skill change (it does report the inherited defects fixed below, as
intended).

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
  partial. The content is what the origins produced and consumed; consolidating it is not a
  behaviour change.
- Two rules the origins left implicit are stated: volatile facts (generated automation, task
  source, implemented phases, blockers) live only in `MEMORY.md` *Current State*, while AGENTS.md
  *Current Reality* and CLAUDE.md *Current Project State* stay durable and point there; and the
  AGENTS.md *Auto-Develop Policy* must declare the pipeline's rollback exception when *Prohibited
  Actions* forbid `git reset --hard` / `git clean`, otherwise the generated script contradicts the
  governance (`[GOVERNANCE DRIFT]`).

### Moved (origin part → new location)
- `prd-to-governance` `SKILL.md` header, file table, generation order, Steps 0 to 8 and 10 to 11,
  adaptation guidelines, quality checklist → `references/govern.md`. Its "Modes" section became
  govern's sub-modes Generate and Update/merge.
- `prd-to-governance` `SKILL.md` Step 9 (audit process, drift categories, audit targets, contract
  coherence checks) → `references/audit.md` Part A. `govern.md` Step 9 links to it.
- `prd-to-governance` `SKILL.md` "Uncertainty Markers" and "Priority Levels" →
  `references/contract.md` sections 1 and 2.
- `prd-to-governance` `references/soul-template.md`, `agents-template.md`, `claude-template.md`,
  `memory-template.md`, `completed-phases-template.md` → `references/` (verbatim apart from the
  cross-skill pointers, which now say "the automate mode", and the two additions listed under
  *Changed*).
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
  `task-list-template.md`, `extraction-checklist.md` → `references/` (verbatim apart from the
  pointer rewrites: "SKILL.md Step N" now reads "`automate.md` Step N", "route back to
  prd-to-governance" now reads "switch to govern mode", and the two pointers to the origin's
  *Memory discipline* and *Deterministic skill resolution* sections now point to `contract.md`
  sections 3 and 4; plus the fixes listed under *Fixed*).
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
- `references/agents-template.md`, `references/claude-template.md`: the *Prohibited Actions* example
  names the pipeline's rollback as the sole exception to the reset/clean ban, the *Auto-Develop
  Policy* example declares it, and the *Current Reality* / *Current Project State* placeholders point
  to MEMORY.md for volatile facts. A behaviour test of the merged skill produced a project whose
  AGENTS.md forbade what its generated script does on failure, and whose status sentences were stale
  right after the automate run.
- `references/govern.md`, `references/automate.md`, `references/audit.md`: govern names both rules
  when writing AGENTS.md/CLAUDE.md; automate checks the prohibition against the rollback in Step 1
  and reports stale status sentences after writing (Step 7); audit has two matching targets, and its
  safety list flags critical pipeline steps that rely on `set -e` instead of explicit guards;
  automate's failure-path wording distinguishes pre-checkpoint (discard) from post-checkpoint (keep
  the committed branch) rollbacks.

### Fixed (deliberate deviation from PRD R6 to R8 and acceptance criterion 5)
The PRD asked for the origin blueprints to be carried over unchanged. Three independent reviews and
a behaviour test reproduced the defects below in the inherited pipeline template, so they are fixed
in 1.0.0 rather than shipped for the sake of parity; the decision is recorded in `docs/PRD.md`
section 6. No origin rule was dropped (`docs/parity.md`).
- `references/auto-develop-template.md`: runtime defects inherited unchanged from
  `governance-to-automation` 1.2.2. They share one cause: bash suspends `set -e` inside a function
  called from an `&& ... ||` list (and never inherits it into `$(...)`), which the template relied
  on. Every critical step of `process_issue` now carries an explicit guard:
  - A failed checkpoint commit, final amend, push, or PR creation fell through to "Done" and was
    counted as a completed issue; a failed checkpoint additionally let the refactor revert land on
    the base tip and discard the approved correctness work. The checkpoint guard logs git's own
    reason, saves the approved diff to `$logdir/approved-uncommitted.patch`, then discards; the
    memory step, the final amend, the push, and the PR keep the committed issue branch, drop only
    leftovers, return to the base branch, and fail the issue.
  - `run_review` accepted any line starting with `LGTM` anywhere in the reviewer output and ignored
    the runner's exit status, so a rejection such as "1. HIGH ..." followed by "LGTM must not be
    granted" passed. It now requires a runner exit status of 0, writes the runner's stderr to a
    `.stderr` sidecar instead of merging it, and decides on the first decisive line: after stripping
    markdown decoration, the first line that is `LGTM` or a numbered finding must be `LGTM` alone or
    `LGTM` followed by a separator (`. ! : ; , ( -`). Reviewer runners must therefore emit only the
    model's final message on stdout (`claude -p --output-format text` does; `codex exec` needs
    `--output-last-message <file>` or `--json`); see the runner contract in the template.
  - A crashed reviewer or fix runner was indistinguishable from "nothing left to fix": the no-op fix
    cycle then accepted the open findings and the issue shipped unreviewed. A crashed reviewer is
    retried once after 15 s and then fails the issue; a crashed fix runner fails the issue.
  - The issue-branch checkout ran inside a `$(...)`, so a failed checkout was swallowed and the run
    continued (and committed) on the base branch. `create_issue_branch` now takes the branch name,
    only checks out, and is guarded; a leftover branch without own commits is recreated from the
    current base tip, one with commits is reused with a warning.
  - The `gh` reads of title, body, and labels in `process_issue` were unguarded and ran the whole
    pipeline with empty values; they now fail the issue before any git mutation.
  - A failed refactor fold (`git commit --amend`) left the round uncommitted for a later revert to
    drop while the counters still reported it; the round is now reverted before the counters move.
  - `check_dependencies` treated an unreadable issue body (failed `gh` call) as "no dependencies"
    and let the task through, and its parser captured only the first `#N` of a `Depends on` line. It
    now fails closed when the body or a dependency's state cannot be read and parses every `#N` on
    the line (`Depends on #12, #13`, `Depends on: #12`), which makes multi-dependency lines comply
    with contract rule M5.
- `references/prompt-builders.md`: the description of the pass rule matches the new check.

The fixture `examples/auto-develop.payload-sample.sh` is a pre-refactor snapshot and is intentionally
not re-synced; it still shows the origin behaviour, including the defects fixed above.

[1.0.0]: https://github.com/Karlderkarl/PRD-to-automation/releases/tag/v1.0.0
