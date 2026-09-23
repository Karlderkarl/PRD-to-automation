# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Changes to `skills/prd-to-automation/references/contract.md` are listed under a **Contract**
heading and bump the contract version stated in that file.

## [Unreleased]

## [1.2.0] - 2026-09-23

Closes the three gaps the 1.1.0 re-test left open: the prompt builders were the largest hand-written
part of every generated script, single review meant deleting code in about nine places, and
reviewers were read-only only by instruction.

### Contract
- `references/contract.md` is now **contract version 1.2.0**.
- M7: reviewers are read-only independent of `--unattended` (a `claude` reviewer without edit and
  shell tools, a `codex` reviewer with `--sandbox read-only`); a review that changes the working
  tree, `MEMORY.md` included, fails the task. `--unattended` widens only the implementer.
- M5: a cross-repository dependency (`owner/repo#N`) blocks the task instead of being read as the
  local `#N`.
- Section 4 states that a `title:` regex is tested against title, newline, and body, so `^` anchors
  the title and `$` the end of the body (clarification, no behaviour change).
- Section 3 no longer cites a "~20,000-character context injection limit", which no Claude Code
  documentation backs; the 15,000-character threshold for `MEMORY.md` is named as the skill's own
  budget (it is imported into every session). The same wording is fixed in `audit.md` and
  `memory-template.md`.
- Field table: a generic model name in governance ("Claude", "Codex") covers every model of that
  provider, so picking `opus` where governance says "Claude" is not drift; only a different
  provider, or a different model than a concrete name, is.

### Added
- `references/prompt-builders.md`: a bash blueprint for all seven prompt builders, inserted
  verbatim by the generator and filled through three placeholders (`{{PROJECT_NAME}}`,
  `{{REFERENCE_DOCS}}`, `{{GOVERNANCE_REVIEW_FOCUS}}`). Untrusted values (title, body, diff,
  findings, check output) reach the prompt only as `printf` arguments or via `cat`, never through an
  unquoted heredoc, so backticks or `$(...)` in a diff are written literally. The earlier prose
  templates are replaced by a description of what each prompt must achieve; the wording lives only
  in the blueprint.
- `references/auto-develop-template.md`: `REVIEW_B_ENABLED` switches between single and dual
  review. The Reviewer B code stays in the script and is skipped; `--review-b` is rejected with a
  message while it is off.

### Changed
- The conflict-resolution order asked in govern's interview (question 10) now has a home: AGENTS.md
  *Conflict Resolution* (`agents-template.md`), which govern's Update/merge strategy follows; audit
  notes its absence as Advisory.
- `run_review` compares a hash of the whole tree (`git write-tree`, `MEMORY.md` included) before and
  after each reviewer; before, a reviewer's edit to `MEMORY.md` went unnoticed.
- The review prompt states one output format, and the pass rule matches it: `LGTM` alone (optionally
  with `ADVISORY:` lines) or numbered findings, never both. A reply with `LGTM` and a numbered line
  now fails; before, it passed when `LGTM` came first. A pass that happens to contain a line such as
  `2024. …` now costs one fix round.
- `check_dependencies` blocks on issue or PR URLs in a `Depends on` line; before, they read as "no
  dependencies".
- `references/automate.md`, `references/extraction-checklist.md`, `SKILL.md`, `README.md`,
  `SECURITY.md`: single review is `REVIEW_B_ENABLED=false`, the builders come from the blueprint, and
  reviewers stay read-only under `--unattended`. `references/audit.md` flags a reviewer runner with
  write or shell access and a tamper check that ignores `MEMORY.md`.

## [1.1.0] - 2026-09-23

A review with an end-to-end test (govern, then automate with a local task list, then audit, on a
sample Python project) and a static test harness for the pipeline template found further runtime
defects, contradictions between the mode references, and test-gate rules that were too loose. All
of them are fixed here; the decision is recorded in `docs/PRD.md` section 6. The safe permission
default stays as it is; the skill now explains what it means for headless runs instead.

### Contract
- `references/contract.md` is now **contract version 1.1.0**.
- `{TARGET}` must not start with `-` or `#` and is substituted shell-quoted (`printf %q`). Before,
  `#` commented out the rest of the command (so `pytest #` ran the whole suite), a leading `-` was
  read as an option, and `[`/`]` were glob-expanded. `{TARGET}` must stand unquoted in
  `TARGETED_TEST_CMD`; the script dies at startup on a quoted one.
- Exit status 126 or 127 (not executable, command not found) is never accepted as RED. Before, a
  missing test runner produced "RED OK" and a GREEN that could never pass.
- Test eligibility base default: any declared `include` matcher, even a dead one, keeps the
  allowlist base; the denylist base needs only `except` matchers, at least one usable. Before, one
  regex typo in an `include` turned testing on for almost every task. A `label:` matcher with an
  empty pattern is dead.
- New generator setting `TASK_SOURCE_HAS_LABELS`: on a local task list or MEMORY.md "Next Up"
  every `label:` matcher (skill and test) is dead and warned at runtime.
- `TARGETED_TEST_CMD` has a defined form (one line `TARGETED_TEST_CMD='<command>'` in the
  *Development Commands* block) and may be marked `# planned`; a planned one counts as absent.
- M7 is enforced at runtime: the pipeline checks before every commit that `SOUL.md`, `AGENTS.md`,
  `CLAUDE.md` are unchanged and fails the task otherwise; a reviewer that changes the working tree
  fails the task. The local task list's `status:` is flipped by the script, never by a model.
- `resolve_skill` warns about and logs entries without `=`, with an empty skill name, or with an
  unknown type, instead of skipping them silently.
- automate's generation-time write scope names what the mode already did: the *Current State* line
  and, when drift is open, one *Governance Drift* line in `MEMORY.md`, plus one entry in
  `memory/completed-phases.md`. Open drift belongs in *Governance Drift*, never only in the archive.
- govern writes M1 to M7 into AGENTS.md *Auto-Develop Policy* (and the status-line and archive
  rules into MEMORY.md *Update Rules*); the section table said MEMORY.md, the template had neither.
- Field table: a user's deliberate model or task-source pick is generated as chosen and recorded as
  open drift; new row *Task source*.

### Fixed
- `references/auto-develop-template.md`:
  - The tmux re-exec crashed for single-review projects (`REVIEW_B_MODEL: unbound variable`) and
    ran a bare `$0`, which is not a command when the script was started as `bash auto-develop.sh`.
    It now re-execs `bash <absolute path>`.
  - A failing `gh issue list` read as "No eligible issues." with exit 0; it now dies.
  - Without `--auto-merge` the base branch was never refreshed, so a dependency merged on GitHub
    counted as done while its code was missing locally. `refresh_base` fast-forwards the base
    before the run and before each issue (never forced; no upstream is a no-op).
  - A branch with own commits (an earlier run failed after the checkpoint) was reused, which made
    `required` tasks fail forever with NOT RED; an issue with an open PR was selected again on every
    run. Both are now skipped as "awaiting review" with a log naming the branch.
  - Started from a subdirectory, the review diff and the clean-worktree guard saw only part of the
    tree. The script now `cd`s to the repository root.
  - A crashed refactor runner was read as "converged"; it now reverts the round. A crashed
    check-fix runner logs a warning.
  - `--dry-run` counted "would process" as completed; it now says "Dry run: would process N".
  - Flags without a value died with an unbound-variable error; `--issue N` did not check that the
    issue is open; `--refactor` was missing from the usage text; `run_checks` leaked `cmd`.
  - A generated script produced six shellcheck findings from template code (unused `*_RUNNER`
    variables, two false positives). The runner variables now back a PATH preflight and are in the
    placeholder legend; the false positives carry a targeted disable with the reason.
  - The skeleton names the skill and contract version in a header comment, so audit can tell
    scripts generated before 1.1.0.
- `references/task-list-template.md`: the local task-list variant existed only as prose, and
  `task_mark_status`, named in the template, was defined nowhere. It now has a bash blueprint
  (parsing, selection, dependency check against the base branch, script-owned status flip,
  awaiting-review skip) and one documented `depends on:` form. A dry run before the first commit
  reads the uncommitted task file and says so, so automate's Step 6 can validate. The issue residue
  to remove in this variant (`--auto-merge`, issue wording) is listed. `status: doing` is gone;
  nothing wrote it.
- `references/prompt-builders.md`: the memory prompt forbids flipping the task status.

### Changed
- `SKILL.md` and `references/govern.md`: a PRD in a documentation folder (`docs/PRD.md`, the
  skill's own example) inside a git repository puts the project root at the repository top level.
  Before, the rule made `docs/` the root.
- `SKILL.md`: automate loads `references/audit.md` in Sync (it needs the drift classes); new
  boundary on headless permissions.
- `references/automate.md`:
  - Two kinds of drift: blocking drift stops generation, a user's deliberate choice (model, task
    source) is generated and recorded as open drift for govern. Before, the text said both "stop"
    and "flag and continue".
  - The approval to write is asked in Step 3, before Steps 4 to 6 write and validate the artifacts.
    Before, Step 7 asked for approval after the files were already written and dry-run.
  - The remaining model steps (test authoring, check-fix, refactor, memory) are shown with the model
    they use.
  - The run guide must explain the headless permission allowlist, committing before the first run,
    that dependent tasks wait for a merge, and `*.sh eol=lf` on Windows.
- `references/govern.md`: Steps 5 to 8 draft, Step 11 writes (`MEMORY.md` and the archive last, so
  "Governance files drafted" is written with them); missing PRD items are marked only when relevant
  to the project; asks for the eligibility label and flags GitHub Issues without a remote; no
  default package manager outside Node; SOUL.md up to 90 lines instead of at least 60.
- `references/audit.md`: one root cause is reported once (a task-source decision is not four drift
  findings); the command check no longer flags `# planned` commands and no longer names
  `package.json`; new targets for stale `# planned` markers, a `TARGETED_TEST_CMD` whose tool is
  missing, markers inside binding rules, and a self-contradicting MEMORY.md; the dry-run check
  covers untracked files and exercises dependency blocking; new safety and advisory findings for
  the rules above.
- `references/agents-template.md`: the *Auto-Develop Policy* example states M3 (archive, not
  inline), M6, M7, the refactor pass, and single review; the MEMORY.md field list matches the
  memory template.
- `references/claude-template.md`: protects `CLAUDE.md` itself; shows the `TARGETED_TEST_CMD` line.
- `references/memory-template.md`: *Update Rules* carry the status-line and archive rules and keep
  open drift in *Governance Drift*; dropped the `memory_search` / `memory_get` tools, which Claude
  Code does not have.
- `references/extraction-checklist.md`: follows the new base-default and `TARGETED_TEST_CMD` rules;
  the branch pattern comes from AGENTS.md; audit uses it too.
- `README.md`: a section on permissions for headless runs; the first run commits before the dry
  run. `SECURITY.md`: the `{TARGET}` quoting; 1.1.x is the supported line.

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

[1.2.0]: https://github.com/Karlderkarl/PRD-to-automation/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/Karlderkarl/PRD-to-automation/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/Karlderkarl/PRD-to-automation/releases/tag/v1.0.0
