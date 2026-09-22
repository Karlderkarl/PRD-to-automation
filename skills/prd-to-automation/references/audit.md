# audit mode: governance drift, script drift, validation

This mode is **read-only**. It writes nothing: no governance file, no script, no `MEMORY.md` line. It combines the audit half of `prd-to-governance` 1.2.0 (governance drift), the Audit/Sync report and Validate mode of `governance-to-automation` 1.2.2 (script drift, validation), and presents them as **one report**. Changes happen only after switching to the govern or automate mode and after the user's approval.

Shared vocabulary and the automation-contract fields: see `references/contract.md`.

For this mode, one project equals one folder. The project root is the folder that contains the governance files; confirm it if ambiguous. Read only inside it.

## Inputs

1. The four governance files `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `MEMORY.md` (plus `memory/completed-phases.md` if present)
2. The PRD, if present
3. The actual repository structure and config (build files, lockfiles, CI, app and test directories)
4. `auto-develop.sh` and its task source, if present

Missing inputs are findings, not errors: no PRD means Part A compares governance to the repository only; no `auto-develop.sh` means Part B and Part C are skipped and reported as "no automation present". Governance that is entirely missing is `[NEEDS GOVERNANCE]` with a recommendation to switch to govern.

A project whose governance came from `prd-to-governance` 1.2.0 and whose script came from `governance-to-automation` 1.2.2 shows no drift merely because the skill changed; only real mismatches are reported.

## Part A: Governance drift

Identify mismatches in four categories:

- **Missing** - governance says something should exist, but it does not
- **Outdated** - governance reflects a past state that is no longer true
- **Contradicted by codebase** - the repo clearly does something different
- **Needs user decision** - the mismatch is strategic, not safely inferable

Tag significant conflicts with `[GOVERNANCE DRIFT]`. Report each finding in one of three forms: "PRD says X, governance says Y", "governance says X, repository does Y", or "governance is silent on X".

Good audit targets include:

- role mismatches between `AGENTS.md` and `CLAUDE.md`
- commands in `CLAUDE.md` that do not exist in `package.json` or other build files
- phase plans that no longer reflect repository reality
- stack declarations in `SOUL.md` that the repo contradicts
- `MEMORY.md` current state or next steps that are stale
- MEMORY.md exceeding ~15,000 characters, a buffer below the ~20,000-character context injection limit (suggest archive split)
- MEMORY.md containing inline completed issue entries instead of only the archive reference
- `memory/completed-phases.md` missing despite a MEMORY.md archive reference
- For projects wired to the automate mode, governance-side coherence of the automation contracts (`references/contract.md` sections 4 to 6):
  - AGENTS.md *Skill Policy* matchers that are malformed (not `<type>:<pattern>=<skill>`, or a `=` inside the pattern), use an unknown type, or overlap so two matchers resolve to different skills for one task (the pipeline would log `(ambiguous)` and inject nothing)
  - AGENTS.md `TEST_POLICY=required` with no `TARGETED_TEST_CMD` in CLAUDE.md -> `[GOVERNANCE DRIFT]` (the pipeline silently degrades to `preferred`)
  - A non-`off` `TEST_POLICY` with empty or inert `TEST_ELIGIBILITY`, or a `TARGETED_TEST_CMD` present while `TEST_POLICY` is `off`/absent -> `[NEEDS GOVERNANCE]` (partial/contradictory test contract)
  - `TARGETED_TEST_CMD` missing its literal `{TARGET}` token
  - `label:` eligibility/skill matchers on a label-less task source (local task-list / MEMORY.md "Next Up"), where only `title:` matchers can ever match

## Part B: Script drift

Read the existing script, re-extract the parameters from the current governance exactly as automate Step 1 does (`references/extraction-checklist.md`), and report mismatches in six classes, tagging significant ones `[GOVERNANCE DRIFT]`:

- **Stale checks** - script runs commands that no longer exist in CLAUDE.md (or misses new ones)
- **Role/model drift** - script roles, runners, or model selections differ from AGENTS.md/CLAUDE.md expectations
- **Memory-rule drift** - script no longer matches MEMORY.md update rules (diff exclusion, status-line, archive ownership, no-op detection; `references/contract.md` section 3)
- **Convention drift** - base branch, label, commit format, or dependency handling diverged
- **Skill-policy drift** - `SKILL_MAP` no longer matches the AGENTS.md *Skill Policy* plus explicitly user-approved local entries (stale/missing matchers, entries with neither a policy nor a user-approval source, or overlapping rules that now resolve to `(ambiguous)`)
- **Test-policy drift** - the script's `TEST_POLICY`, `TEST_ELIGIBILITY`, or targeted-test gate no longer matches AGENTS.md *Auto-Develop Policy* / CLAUDE.md `TARGETED_TEST_CMD`

Also report these governance contract gaps explicitly:

- `[GOVERNANCE DRIFT]` when AGENTS.md sets `TEST_POLICY=required` but CLAUDE.md declares no `TARGETED_TEST_CMD` (deterministic gate unavailable → degrade to `preferred`)
- `[NEEDS GOVERNANCE]` when test fields are partial or contradictory: `TARGETED_TEST_CMD` exists while `TEST_POLICY` is off/absent, or `TEST_POLICY` is set but `TEST_ELIGIBILITY` is empty/inert. Test fields entirely absent are the valid default (`off`), not drift.

Check the script's safety and invariants as well, and report deviations as **Critical** findings:

- privileged values (`bypassPermissions`, `danger-full-access`, an unconditional merge) appear as defaults instead of behind `--unattended` / `--auto-merge` and `confirm_privileged_mode`
- `eval` is used on project commands or on the `{TARGET}` value, or the `{TARGET}` allowlist sanitisation is missing
- the checkpoint commit is not gated on a non-empty code diff; failure paths use a bare `git checkout` instead of discarding work and returning to the base branch
- `FROZEN_TARGETED_TEST_TARGET`, `TARGETED_TEST_FILE`, `TEST_GATE_ACTIVE` are not reset per task
- the refactor pass runs before the correctness checkpoint, or keeps a round whose re-review was not clean

## Part C: Validation

Validate the script without running the real loop:

1. `bash -n auto-develop.sh` (syntax)
2. `shellcheck auto-develop.sh`, if available; report findings honestly, including "shellcheck not installed"
3. `./auto-develop.sh --dry-run`, if the environment allows, to confirm candidate selection works without executing models. `--dry-run` must be side-effect-free; if it mutates tracked files, that is a **Critical** finding

Never execute the real pipeline loop as part of an audit.

## The report

Present one report with three parts, in this order:

1. **Governance drift** (Part A): findings by category, each with its marker and the form "X says, Y does"
2. **Script drift** (Part B): findings by drift class, then contract gaps, then safety findings; or "no automation present"
3. **Validation** (Part C): the exact commands run and their results; or "not run" with the reason

Then:

- summarize the most important mismatches and every `[NEEDS PRD CLARIFICATION]`, `[NEEDS CODEBASE DISCOVERY]`, `[USER DECISION REQUIRED]`, `[GOVERNANCE DRIFT]`, and `[NEEDS GOVERNANCE]` item
- propose the mode switch that would resolve each finding: governance-side findings go to the **govern** mode (Update/merge), script-side findings go to the **automate** mode (Sync)
- ask for explicit approval before switching; the switch itself writes nothing until the target mode has presented its proposal and the user has approved it

## Quality checklist

Before presenting the report:

- [ ] Mode stated (`audit`) and no file was written
- [ ] All available governance files, the PRD, and the repository were read before any claim about commands, stack details, or phase status
- [ ] Missing inputs are reported as findings, not silently skipped
- [ ] Every governance finding is classified (Missing / Outdated / Contradicted by codebase / Needs user decision) and significant conflicts carry `[GOVERNANCE DRIFT]`
- [ ] Automation-contract coherence was checked on the governance side (matcher form, `TEST_POLICY` versus `TARGETED_TEST_CMD`, `{TARGET}` token, label-less sources)
- [ ] Script findings are sorted into the six drift classes plus contract gaps plus safety invariants
- [ ] Validation results name the exact commands and their outcome; the real loop was never run
- [ ] No drift was reported merely because the governance or script came from the origin skills
- [ ] The report ends with proposed mode switches and asks for approval instead of making changes
