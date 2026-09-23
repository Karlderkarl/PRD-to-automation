# Parity: origin rules and their new location

Release blocker: every rule below must have a location. A rule with no location cannot ship.

1.0.0 additionally fixes runtime defects of the origin pipeline template and adds two template rules (CHANGELOG, *Fixed* and *Changed*); no origin rule was dropped or weakened, so every row below still holds.

1.2.0 changes no row: it adds the prompt-builder blueprint, the `REVIEW_B_ENABLED` toggle, and strictly read-only reviewers (CHANGELOG 1.2.0, contract 1.2.0).

1.1.0 tightens rules and fixes further defects (CHANGELOG 1.1.0, contract 1.1.0). No row lost its location. Two rows changed meaning and say so: A21 and R27 (automate additionally writes one archive entry and, when drift is open, one *Governance Drift* line; the origin already sent the detail to the archive, 1.1.0 makes the boundary say it), and A4/A26 (a user's deliberate model or task-source pick is generated as chosen and recorded as open drift for govern, instead of stopping generation).

Sources: `prd-to-governance` 1.2.0 `SKILL.md` "Quality Checklist" (P1 to P15), `governance-to-automation` 1.2.2 `SKILL.md` "Quality checklist" (A1 to A26), and the Critical invariants of PRD section 4.4 (R14 to R27). Paths are relative to `skills/prd-to-automation/`, except `SECURITY.md` and `examples/`, which sit at the repository root.

## prd-to-governance 1.2.0, Quality Checklist

| # | Origin rule | New location |
|---|---|---|
| P1 | Mode selection was stated clearly (Generate vs Audit) | `SKILL.md` "Mode selection" (state the mode in one line); `references/govern.md` Step 0 and checklist item 1 (Generate vs Update/merge); `references/audit.md` checklist item 1 |
| P2 | Project root derived from the PRD location or confirmed by the user | `SKILL.md` "Project root"; `references/govern.md` Step 1 and checklist item 2 |
| P3 | Existing governance files handled per user decision (overwrite / merge / skip) or treated as audit inputs | `references/govern.md` Step 1 and checklist item 3; `SKILL.md` mode-selection row 2 |
| P4 | The PRD was read completely if one was provided | `references/govern.md` Step 2 and checklist item 4; `references/audit.md` Inputs and checklist item 2 |
| P5 | Repository reality was inspected before claiming commands, stack details, or phase status | `references/govern.md` Step 3 and checklist item 5; `references/audit.md` checklist item 2 |
| P6 | SOUL.md contains only non-negotiable principles | `references/govern.md` Step 5 and checklist item 6 |
| P7 | AGENTS.md prohibited actions are specific and enforceable | `references/govern.md` Step 6 and checklist item 7 |
| P8 | CLAUDE.md `@references` point to the correct filenames | `references/govern.md` Step 7 and checklist item 8 |
| P9 | CLAUDE.md commands match actual repo config or are marked `# planned` | `references/govern.md` Step 7 and checklist item 9 |
| P10 | MEMORY.md preserves historical state and records governance drift | `references/govern.md` Steps 8 and 10, checklist item 10; `references/contract.md` section 3 (layout) |
| P11 | Automation-contract fields present are coherent across files and use the exact `<type>:<pattern>=<value>` form; absent fields left absent | `references/govern.md` Steps 4, 6, 7 and checklist item 11; `references/contract.md` sections 4 to 6; `references/audit.md` Part A contract coherence |
| P12 | No secrets, passwords, or API keys appear in any file | `references/govern.md` checklist item 12 |
| P13 | All files are consistent with each other (roles, phases, repo status) | `references/govern.md` checklist item 13; `references/audit.md` Part A audit targets (role mismatches) |
| P14 | Drift or conflicts are marked explicitly, never hidden or guessed away | `references/govern.md` Step 10 and checklist item 14; `references/contract.md` section 1; `references/audit.md` Part A |
| P15 | Governance overhead is proportional to project complexity | `references/govern.md` "Adaptation Guidelines" and checklist item 15 |

Other `prd-to-governance` sections that are not checklist items:

| Origin section | New location |
|---|---|
| Uncertainty Markers, incl. the `[NEEDS GOVERNANCE]` scoping rule | `references/contract.md` section 1 |
| Priority Levels | `references/contract.md` section 2 |
| Step 9 Audit Mode (process, four categories, audit targets, contract coherence) | `references/audit.md` Part A; `references/govern.md` Step 9 links to it |
| Step 10 Merge Strategy | `references/govern.md` Step 10 |
| Step 11 Present and Confirm (write only after approval, record in memory after the write, do not commit) | `references/govern.md` Step 11; `SKILL.md` "Presenting work"; `references/contract.md` section 3 (recording after the write) |
| Adaptation Guidelines | `references/govern.md` "Adaptation Guidelines" |
| Five templates | `references/soul-template.md`, `agents-template.md`, `claude-template.md`, `memory-template.md`, `completed-phases-template.md` |

## governance-to-automation 1.2.2, Quality checklist

| # | Origin rule | New location |
|---|---|---|
| A1 | Mode stated (Generate / Audit-Sync / Validate) | `SKILL.md` "Mode selection"; `references/automate.md` "Sub-modes" and checklist item 1 (Generate / Sync); Validate is `references/audit.md` Part C |
| A2 | All four governance files read; missing ones blocked the run with `[NEEDS GOVERNANCE]` | `references/automate.md` Step 0 and checklist item 2; `SKILL.md` boundaries and mode-selection row 6 |
| A3 | Checks, base branch, task source extracted from governance, not guessed | `references/automate.md` Step 1 and checklist item 3; `references/extraction-checklist.md` |
| A4 | Models explicitly chosen by the user per step; divergence from governance flagged `[GOVERNANCE DRIFT]` | `references/automate.md` Step 3 item 1 and checklist item 4; `SKILL.md` boundaries; `references/contract.md` section 6 (Roles and models) |
| A5 | Exactly one task source is wired | `references/automate.md` Step 2 and checklist item 5 |
| A6 | Each issue branches from the base branch and returns to it; failure paths discard work, never a bare `git checkout` | `references/automate.md` Step 4 and checklist item 6; `references/auto-develop-template.md` |
| A7 | Checkpoint commit gated on a non-empty code diff | `references/automate.md` Step 4 and checklist item 7; `references/contract.md` M6; `references/audit.md` Part B safety |
| A8 | Backlog materialized from the AGENTS.md Phase Plan into the chosen source | `references/automate.md` Step 5 and checklist item 8; `references/task-list-template.md` |
| A9 | Usage covers detached long runs (`tmux`) when asked | `references/automate.md` Step 3 item 5, Step 5, checklist item 9 |
| A10 | Every memory-discipline rule present in the generated script | `references/contract.md` section 3 (M1 to M7); `references/automate.md` "Memory discipline", Step 4, checklist item 10 |
| A11 | Skill resolution deterministic: `SKILL_MAP` from governance or approved local entries; `resolve_skill` once per task, logged; `(ambiguous)` injects nothing; injection only into implement/fix/refactor | `references/contract.md` section 4; `references/automate.md` "Deterministic skill resolution", Step 4, checklist item 11 |
| A12 | Test policy wired for the chosen task source; per-task reset of `TARGETED_TEST_FILE`, `FROZEN_TARGETED_TEST_TARGET`, `TEST_GATE_ACTIVE`; `label:` on label-less sources is `[GOVERNANCE DRIFT]` | `references/contract.md` sections 5 and 6; `references/automate.md` checklist item 12; `references/task-list-template.md`; `references/audit.md` Part B safety |
| A13 | Test policy deterministic and opt-in: absent is `off`; explicit include/except matchers; `except` wins; no ambiguous outcome; inert set is `off` and warned | `references/contract.md` section 5 and section 6; `references/automate.md` checklist item 13 |
| A14 | Gate proves red→green with honest scope; RED required before implementation; frozen target; not exit-verified; hard gate only under `required`; `preferred` advisory | `references/contract.md` section 5; `references/automate.md` checklist item 14 |
| A15 | Model-authored `{TARGET}` sanitized before substitution; `bash -c`, not `eval` | `references/contract.md` section 5; `references/automate.md` checklist item 15; `references/audit.md` Part B safety; `SECURITY.md` |
| A16 | Gate scope stated honestly: targeted TDD gate, not a no-regression gate; baseline out of scope for v1 | `references/contract.md` section 5; `references/automate.md` checklist item 16; `references/auto-develop-template.md` known limitation |
| A17 | Review enforces the policy asymmetrically (`required` blocks, `preferred` uses `ADVISORY:`) | `references/contract.md` section 5; `references/automate.md` checklist item 17; `references/prompt-builders.md` |
| A18 | `TEST_POLICY=required` needs `TARGETED_TEST_CMD`; otherwise degrade to `preferred` with `[GOVERNANCE DRIFT]` | `references/contract.md` section 6; `references/automate.md` Step 1 and checklist item 18; `references/audit.md` Parts A and B |
| A19 | Refactor pass only after the committed checkpoint, via shared `review_until_pass`, kept only on a clean re-review, bounded by no-op detection and `MAX_REFACTOR_ROUNDS` | `references/automate.md` "Two-pass pipeline", Step 4, checklist item 19; `references/audit.md` Part B safety |
| A20 | Metadata reflects real history: delivered A/B rounds; correctness fixes and refactor rounds distinct | `references/automate.md` "Two-pass pipeline" and checklist item 20; `references/contract.md` M3 |
| A21 | `SOUL.md`, `AGENTS.md`, `CLAUDE.md` not edited; only `MEMORY.md` plus artifacts changed (since 1.1.0 also one archive entry) | `SKILL.md` boundaries; `references/automate.md` "Governance is the contract" and checklist item 21; `references/contract.md` section 3 (generation-time scope and M7) |
| A22 | Prompts instruct agents to read the governance and do not duplicate large governance text | `SKILL.md` boundaries ("link, don't duplicate"); `references/automate.md` Step 4 and checklist item 22; `references/prompt-builders.md` |
| A23 | Privileged flags off by default, only via `--unattended` / `--auto-merge` behind `confirm_privileged_mode`; tmux re-exec propagates opt-in plus `--yes` | `SKILL.md` boundaries; `references/automate.md` Step 3 item 4 and checklist item 23; `references/auto-develop-template.md`; `references/audit.md` Part B safety; `SECURITY.md` |
| A24 | Script passed `bash -n` (and `shellcheck`); `--dry-run` offered | `references/automate.md` Step 6 and checklist item 24; `references/audit.md` Part C |
| A25 | `logs/` ignored and `memory/completed-phases.md` not accidentally ignored | `references/automate.md` Step 5 and checklist item 25; `references/contract.md` section 3 (layout) |
| A26 | Governance mismatches marked `[GOVERNANCE DRIFT]` and routed to governance, not patched into the script | `references/automate.md` "Governance is the contract", Step 3 item 1, checklist item 26 (switch to govern) |

Other `governance-to-automation` sections that are not checklist items:

| Origin section | New location |
|---|---|
| Artifact table ("What this skill produces") | `references/automate.md` "What this mode produces" |
| Governance is the contract (Critical boundary) | `references/automate.md` "Governance is the contract"; `SKILL.md` boundaries |
| Memory discipline is the core integration (the five rules) | `references/contract.md` section 3 M1 to M5; `references/automate.md` keeps the Critical statement |
| Deterministic skill resolution (six invariants) | `references/contract.md` section 4; `references/automate.md` "Deterministic skill resolution" |
| Two-pass pipeline: correctness, then refactor | `references/automate.md` "Two-pass pipeline" |
| Uncertainty and priority markers | `references/contract.md` sections 1 and 2 |
| Modes (Generate / Audit-Sync / Validate) and default selection | `references/automate.md` "Sub-modes"; `references/audit.md`; `SKILL.md` mode selection |
| Steps 0 to 7 | `references/automate.md` Steps 0 to 7 |
| Audit/Sync mode (six drift classes, contract gaps) | `references/audit.md` Part B; `references/automate.md` "Sync" |
| Adaptation guidelines | `references/automate.md` "Adaptation guidelines" |
| Four blueprints | `references/auto-develop-template.md`, `prompt-builders.md`, `task-list-template.md`, `extraction-checklist.md` |
| Examples | `examples/auto-develop.payload-sample.sh`, `examples/refact-todo.md` |

## PRD section 4.4, Critical invariants

| PRD | Invariant | New location |
|---|---|---|
| R14 | Review diffs exclude `MEMORY.md` | `references/contract.md` M1; `references/agents-template.md` Auto-Develop Policy; `references/auto-develop-template.md` |
| R15 | Implement and fix steps write exactly one overwritten "Next Up" line | `references/contract.md` M2; `references/prompt-builders.md` |
| R16 | Only the post-review memory step writes completed work, to `memory/completed-phases.md` | `references/contract.md` M3; `references/prompt-builders.md` |
| R17 | No-op fix detection breaks the loop | `references/contract.md` M4; `references/auto-develop-template.md` |
| R18 | `Depends on #N` hard-blocks | `references/contract.md` M5; `references/auto-develop-template.md`; `references/task-list-template.md` |
| R19 | Checkpoint commit requires a non-empty code diff; no commit or PR on a memory-only run | `references/contract.md` M6; `references/automate.md` Step 4; `references/auto-develop-template.md` |
| R20 | Skill resolution only via explicit `label:` / `title:` matchers, once per task, logged; `(ambiguous)` injects nothing; injection only into implement/fix/refactor | `references/contract.md` section 4; `references/automate.md` "Deterministic skill resolution" |
| R21 | Test resolution with the same discipline; absent is `off`; `except` beats `include`; no ambiguous outcome; inert is `off` with warning | `references/contract.md` section 5; `references/automate.md` checklist item 13 |
| R22 | Gate proves red→green for one named test; not a regression gate; blocks only under `required` | `references/contract.md` section 5; `references/automate.md` checklist items 14 and 16 |
| R23 | `{TARGET}` allowlist-sanitised before `bash -c`; no `eval` | `references/contract.md` section 5; `SECURITY.md`; `references/audit.md` Part B safety |
| R24 | Privileged modes off; only via `--unattended` / `--auto-merge` behind `confirm_privileged_mode`; never defaults, not in fixtures | `SKILL.md` boundaries; `references/automate.md` Step 3 item 4; `references/auto-develop-template.md`; `examples/`; `SECURITY.md` |
| R25 | No assumed toolchain; `CHECKS=()` verbatim from `CLAUDE.md`; empty is a valid no-op | `SKILL.md` boundaries; `references/contract.md` section 6; `references/automate.md` Step 1 and "Any stack?" |
| R26 | Refactor pass only after the committed checkpoint, via `review_until_pass`, kept only on clean re-review, `--no-refactor`, `MAX_REFACTOR_ROUNDS` | `references/automate.md` "Two-pass pipeline" and checklist item 19 |
| R27 | Automation writes only `MEMORY.md` and generated artifacts (since 1.1.0 also one archive entry); generation never runs the real loop | `SKILL.md` boundaries; `references/contract.md` section 3 (generation-time scope); `references/automate.md` Step 6 and checklist item 21 |
