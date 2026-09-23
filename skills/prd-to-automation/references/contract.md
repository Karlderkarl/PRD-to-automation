# The governance-to-pipeline contract

**Contract version: 1.1.0.** Any change to this file is a contract change and is recorded as such in `CHANGELOG.md`.

This file defines once what the **govern** mode writes, the **automate** mode reads, and the **audit** mode checks. The mode references (`references/govern.md`, `references/automate.md`, `references/audit.md`) and the blueprints link here for the definitions; wherever a mode reference restates a rule for readability, this file is the authoritative wording. Version 1.0.0 was exactly what `prd-to-governance` 1.2.0 produced and `governance-to-automation` 1.2.2 consumed; every later change is listed under a **Contract** heading in `CHANGELOG.md`.

| Section | govern (producer) | automate (consumer) | audit (checker) |
|---|---|---|---|
| 1 Uncertainty markers | uses them in generated files and reports | uses them in reports and script warnings | uses them to tag findings |
| 2 Priority levels | tags rules in governance | tags rules in prompts and reports | reports by level |
| 3 Memory rules | writes M1 to M7 into AGENTS.md *Auto-Develop Policy* and the status-line and archive rules into `MEMORY.md` *Update Rules* | encodes every rule into the generated script | reports memory-rule drift |
| 4 Skill Policy | optional AGENTS.md section | seeds `SKILL_MAP`, resolved by `resolve_skill` | reports malformed or ambiguous matchers, skill-policy drift |
| 5 Test discipline | optional AGENTS.md and CLAUDE.md fields | seeds `TEST_POLICY`, `TEST_ELIGIBILITY`, `TARGETED_TEST_CMD`, resolved by `resolve_test_policy` | reports partial or contradictory fields, test-policy drift |
| 6 Field table | what is optional and what is required | what happens when a field is absent | what counts as a gap |

## 1. Uncertainty markers

Use explicit markers instead of a vague "TBD" whenever possible:

- `[NEEDS PRD CLARIFICATION]`: the PRD is the intended source of truth, but it is incomplete or ambiguous
- `[NEEDS CODEBASE DISCOVERY]`: the answer depends on inspecting the actual repository
- `[USER DECISION REQUIRED]`: the choice is strategic or preference-based and must not be inferred (in automate: models, auto-merge, sandbox level, task source)
- `[GOVERNANCE DRIFT]`: the PRD, the governance files, and the current repository or generated script disagree
- `[NEEDS GOVERNANCE]`: a downstream-automation contract that the governance is expected to define (sections 3 to 5) is missing or too thin to generate from, for example a `TEST_POLICY` set without usable `TEST_ELIGIBILITY` matchers. In govern, use it only for these automation-contract fields; for ordinary gaps prefer the four markers above. In automate, it means: stop, switch to govern, never invent the policy in the script.

## 2. Priority levels

- **Critical**: cannot be violated without explicit user approval; security, compliance, and architectural boundaries
- **Required**: default operating rule; deviations need explanation
- **Advisory**: recommendation or preferred pattern; not blocking

Do not force priority tags onto every bullet. Use them where they clarify what truly matters.

## 3. Memory rules

Layout that govern produces (blueprints: `references/memory-template.md`, `references/completed-phases-template.md`, `references/agents-template.md` *Auto-Develop Policy*):

- `MEMORY.md` is the living state: Current State, Completed Work (archive reference only), Key Decisions, Key Implementation Notes, Next Up, Content Sources, Infrastructure, Governance Drift, Update Rules. Start at 40 to 60 lines.
- `memory/completed-phases.md` is the archive for completed-work details, created by default and organised with `### Phase Name` subheadings. It must not be gitignored; if `memory/` holds daily flush files, ignore them with a precise pattern such as `memory/2026-*.md`.
- Volatile facts (what exists right now: generated automation, the task source, implemented phases, current blockers) live only in `MEMORY.md` *Current State*. AGENTS.md *Current Reality* and CLAUDE.md *Current Project State* describe durable structure and point to `MEMORY.md`; the automate mode may not edit those two files, so any "does not exist yet" sentence there is stale the moment it runs.
- `MEMORY.md` above roughly 15,000 characters is a drift finding (a buffer below the roughly 20,000-character context injection limit); the remedy is an archive split, never deletion.
- "Governance files drafted", a governance audit or update, and "Auto-develop pipeline generated" are recorded only after the corresponding write succeeded, with details in the archive. govern writes `MEMORY.md` and the archive last, in the same approved write, so the entry is added once the other selected files exist; if an earlier file failed, the entry is not written.
- Open drift lives in `MEMORY.md` *Governance Drift*, never only in the archive, which holds completed work.

Rules that every generated pipeline must implement exactly (all **Critical**):

- **M1 Diff exclusion**: review diffs and no-op comparisons exclude `MEMORY.md` (`git diff <base> -- . ':!MEMORY.md'`, staged against the base so new files are included) and the log directory.
- **M2 One status line**: the implement, fix, and refactor steps write exactly one "Next Up" status line to `MEMORY.md`, overwriting, never appending, after reading its Update Rules.
- **M3 Archive ownership**: only the dedicated post-review memory step writes completed work, and it writes to `memory/completed-phases.md`, never inline. It records correctness fix rounds and accepted refactor rounds as distinct facts.
- **M4 No-op fix detection**: if a fix cycle changes only `MEMORY.md` or logs and no real code, the remaining findings are accepted deviations and the review loop breaks.
- **M5 Dependency blocking**: `Depends on #N` (or the task-list equivalent) hard-blocks a task until every dependency is done; blocked tasks are skipped, not failed.
- **M6 Non-empty checkpoint**: the correctness checkpoint commit requires a non-empty code diff (excluding `MEMORY.md` and logs). A memory-only run produces no commit and no PR.
- **M7 Governance is read-only at runtime**: the running pipeline writes application code and tests, `MEMORY.md` and the archive, its own logs under the log directory, and the task source's status (a local task list's `status:` field, flipped by the script, never by a model; a GitHub issue is closed through its PR), and nothing else. Every write-capable prompt forbids editing `SOUL.md`, `AGENTS.md`, and `CLAUDE.md` and forbids committing; the pipeline owns the commit and verifies before every commit that those three files are unchanged against the base branch, failing the task otherwise. Reviewers are read-only: a review that changes the working tree fails the task.

Generation-time scope is a different thing: the automate mode itself writes only `MEMORY.md` (the *Current State* line and, when drift is open, one *Governance Drift* line), one entry in `memory/completed-phases.md`, and the generated artifacts (script, task source, `.gitignore` entry, run guide), and never edits `SOUL.md`, `AGENTS.md`, or `CLAUDE.md`. The mode generates; the pipeline implements.

If the governance does not specify M1 to M5, automate emits `[NEEDS GOVERNANCE]` and switches to govern; it never invents the policy.

## 4. Skill Policy and `SKILL_MAP`

**Producer** (govern, blueprint `references/agents-template.md` *Skill Policy Example*): an optional AGENTS.md section *Skill Policy*. Each line is one explicit matcher `<type>:<pattern> = <skill-name>`:

- `<type>` is `label` (matched against a whole issue or task label; multi-word labels are fine) or `title` (an extended regex tested against the task title and body).
- Whitespace around `:` and `=` is optional. The pattern may contain `:` but never `=`.
- Matchers must be unambiguous: if two matchers resolve to different skills for the same task, the pipeline logs `(ambiguous)` and injects nothing. Keep patterns disjoint.
- Omitting the section is a valid no-op. Never invent matchers to fill it.

**Consumer** (automate, blueprint `references/auto-develop-template.md` `resolve_skill`):

- `SKILL_MAP=()` holds the matchers from AGENTS.md plus any entries the operator authored locally with explicit approval (marked as local in the sign-off). Empty is fully functional.
- `resolve_skill` runs once per task, before implementation, and sets `RESOLVED_SKILL` and `RESOLVED_SKILL_REASON`. It touches no filesystem, registry, or network and performs no semantic search.
- Exactly one distinct match is chosen. More than one distinct match is `(ambiguous)` and nothing is injected. Zero matches is `(none)`. An invalid `title:` regex is logged as an invalid-policy warning, never silently skipped.
- Every decision is written to `$LOGDIR/<task>/skill-resolution.log` as `searched`, `candidates`, `chosen`, `reason`.
- The result is injected only into the implement, fix, and refactor prompts. Reviewers, check-fix, and the memory step stay skill-neutral.
- An entry without `=`, with an empty skill name, or with an unknown type is warned and written to `skill-resolution.log`, never silently skipped.
- On label-less task sources (local task list, `MEMORY.md` "Next Up") only `title:` matchers can resolve. The generator sets `TASK_SOURCE_HAS_LABELS=false` there, and every `label:` matcher is dead and warned at runtime.

## 5. Test discipline

**Producer** (govern, blueprints `references/agents-template.md` *Test discipline*, `references/claude-template.md` *Development Commands*):

- AGENTS.md *Auto-Develop Policy*: `TEST_POLICY` is `off`, `preferred`, or `required`; `TEST_ELIGIBILITY` is one matcher per line in the form `<type>:<pattern>=<include|except>` with the same `label:` / `title:` types as section 4.
- CLAUDE.md *Development Commands*: one line `TARGETED_TEST_CMD='<command>'` inside the commands block, with a literal, unquoted `{TARGET}` token (for example `TARGETED_TEST_CMD='pytest {TARGET}'`); the pipeline quotes the target itself. Include it only when `TEST_POLICY` is not `off`. If the command's tool is not installed yet, mark the line `# planned` like any other planned command; the pipeline then treats it as absent until govern removes the marker.
- Omitting all fields keeps the gate `off`; that is the backward-compatible default, not a gap.

**Consumer** (automate, blueprint `references/auto-develop-template.md` `resolve_test_policy`, `run_targeted_test_gate`):

- Eligibility resolves once per task, logged to `test-policy.log`, with the same discipline as skill resolution and no "ambiguous" outcome: `except` wins over `include`; then an `include` match is eligible; otherwise the base default follows the declared matchers: allowlist whenever any `include` matcher is declared (a dead `include` still counts as declared intent, so a typo can never flip the base to a denylist), denylist only when every declared matcher is `except` and at least one of them is usable. A dead matcher (unknown type, invalid regex, malformed entry, empty pattern, or a `label:` matcher on a label-less source) is warned and never arms the denylist base. An empty or inert set fails safe to `off` with a warning; it never falls through to "test every task". Wiring is task-source-general; on label-less sources only `title:` matchers can match.
- The gate proves a red-to-green transition for exactly one designated test: the model authors the test before implementation, `expect_red` must return non-zero, and the same target is rerun after implementation. The RED-confirmed target is frozen in `FROZEN_TARGETED_TEST_TARGET` for the rest of the task. `TARGETED_TEST_FILE`, `FROZEN_TARGETED_TEST_TARGET`, and `TEST_GATE_ACTIVE` are reset at the top of every task in every variant.
- The hard, blocking gate (`TEST_GATE_ACTIVE`, enforced by `ensure_checks_pass`) is armed only under `required`: an unprovable RED fails the task and a still-red target blocks. Under `preferred` the rerun still happens, but an unresolved failure is advisory; `preferred` never hard-blocks, never discards correctness work, and never triggers extra code mutation when ordinary checks are green.
- Review enforcement is asymmetric: `required` missing tests are blocking numbered findings; `preferred` missing tests use the non-blocking `ADVISORY:` channel (a leading `LGTM` followed by `ADVISORY:` lines still passes).
- The model-authored `{TARGET}` is sanitised against the allowlist `^[][A-Za-z0-9_./:@=+#-]+$`, must not start with `-` or `#`, and is substituted shell-quoted (`printf %q`) into the command, which runs via `bash -c`, never `eval`.
- Honest scope: this is a targeted TDD gate, not a no-regression gate. Exit status 126 or 127 (not executable, command not found) is never accepted as RED; it counts as an unprovable RED. Any other non-zero RED is not exit-verified as an assertion failure (no framework exit-code parsing). Regression coverage is whatever `CHECKS[]` runs.

## 6. Field table

| Field | File and section | Status | When absent | When partial or contradictory |
|---|---|---|---|---|
| *Skill Policy* | AGENTS.md | optional | `SKILL_MAP=()`, valid no-op | malformed matcher, unknown type, `=` in pattern, or overlapping matchers: `[GOVERNANCE DRIFT]` in audit; at runtime `(ambiguous)` injects nothing. `label:` matchers on a label-less source: `[GOVERNANCE DRIFT]` |
| `TEST_POLICY` | AGENTS.md *Auto-Develop Policy* | optional | `off` | unknown value: `[NEEDS GOVERNANCE]` |
| `TEST_ELIGIBILITY` | AGENTS.md *Auto-Develop Policy* | required when `TEST_POLICY` is not `off` | with policy set: inert, so `off` plus warning and `[NEEDS GOVERNANCE]` | dead matchers warned; `label:` on a label-less source: `[GOVERNANCE DRIFT]` |
| `TARGETED_TEST_CMD` | CLAUDE.md *Development Commands* | required when `TEST_POLICY=required` | absent or `# planned`: `required` degrades to `preferred` with a logged `[GOVERNANCE DRIFT]` | present while policy is `off` or absent: `[NEEDS GOVERNANCE]`; missing or quoted `{TARGET}` token: `[GOVERNANCE DRIFT]`; not marked `# planned` although its tool is not installed: `[GOVERNANCE DRIFT]` |
| *Update Rules* (M1 to M5) | MEMORY.md, AGENTS.md *Auto-Develop Policy* | required for automate | `[NEEDS GOVERNANCE]`, switch to govern | any rule contradicted by the script: memory-rule drift |
| *Development Commands* | CLAUDE.md | required source of `CHECKS[]` | `CHECKS=()`, valid no-op; `# planned` commands are not runnable and are excluded | script runs a command not in CLAUDE.md, or misses one: stale-checks drift |
| *Roles* and models | AGENTS.md, CLAUDE.md | suggested default only | user is asked in automate Step 3 regardless | user's pick differs: generated as chosen and recorded as open `[GOVERNANCE DRIFT]`; the user decides whether govern records the choice or automate Sync reverts to the governance |
| Task source | AGENTS.md *Workflow*, MEMORY.md *Key Decisions* | required for automate | `[USER DECISION REQUIRED]` in automate Step 2 | user's pick differs (for example no GitHub remote): handled like a differing model pick; `label:` matchers that only fit the governed source are reported under this one decision, not as separate drift |
| Git conventions | AGENTS.md | required for automate | `[NEEDS GOVERNANCE]` or `[USER DECISION REQUIRED]` | diverged: convention drift |
| *Phase Plan* | AGENTS.md | source of the backlog | task source is scaffolded empty and `[USER DECISION REQUIRED]` is raised | n/a |
| Rollback exception | AGENTS.md *Auto-Develop Policy* | required when *Prohibited Actions* forbid `git reset --hard` / `git clean` | the generated script's failure-path discard contradicts the prohibition: `[GOVERNANCE DRIFT]`, resolved through govern, never by weakening the script | n/a |
