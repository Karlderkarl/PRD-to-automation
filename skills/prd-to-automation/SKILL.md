---
name: prd-to-automation
description: "Turn a PRD into project governance (SOUL.md, AGENTS.md, CLAUDE.md, MEMORY.md), turn that governance into a project-tailored, stack-agnostic auto-develop pipeline script, and audit both for drift. Use when bootstrapping or refreshing governance from a PRD, generating or syncing an auto-develop.sh from existing governance, or checking governance and pipeline drift, including dry-running a generated script. Do not trigger merely because a repository contains an AGENTS.md; the user must ask for governance, automation, or an audit of them."
license: MIT
metadata:
  version: 1.1.0
---

# PRD to Automation

One skill, three modes. It takes a project from a PRD to governance files, from governance to a generated `auto-develop.sh` pipeline, and audits both for drift. Each mode preserves the behaviour of its origin: **govern** is `prd-to-governance` 1.2.0, **automate** is `governance-to-automation` 1.2.2, **audit** joins the read-only audit parts of both. Every deliberate deviation from the origins is listed in the CHANGELOG. Those names appear in this skill only as provenance; never install or invoke them.

## Project root

One project equals one folder. The project root is the folder that contains the four governance files (automate, audit). For govern it is the folder that contains the PRD, except when the PRD sits in a documentation folder such as `docs/`, `docs/prd/`, `planning/specs/`, or `.github/` inside a git repository: then the default root is the repository's top-level folder (`git rev-parse --show-toplevel`), stated in one line before any work. Read and write only inside the root unless the user says otherwise. If the PRD lives outside the real project folder, sits in a documentation folder outside any git repository, or was pasted inline, stop and ask which folder is the project root before writing anything.

## Modes

| Mode | Writes | References to load |
|---|---|---|
| **govern** | `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `MEMORY.md`, `memory/completed-phases.md` | `references/govern.md`, `references/contract.md`; `references/audit.md` when governance files already exist (Update/merge); then `references/soul-template.md`, `references/agents-template.md`, `references/claude-template.md`, `references/memory-template.md`, `references/completed-phases-template.md` as each file is generated |
| **automate** | `auto-develop.sh`, the task source, a `.gitignore` entry, a run guide, at most two lines in `MEMORY.md`, one entry in `memory/completed-phases.md` | `references/automate.md`, `references/contract.md`, `references/extraction-checklist.md`; `references/audit.md` in Sync (drift classes); then `references/auto-develop-template.md`, `references/prompt-builders.md`, `references/task-list-template.md` |
| **audit** | nothing | `references/audit.md`, `references/contract.md`; `references/extraction-checklist.md` when an `auto-develop.sh` is present |

Load only the references the chosen mode needs. The templates and blueprints in `references/` are structural blueprints, not rigid forms; adapt them to the project.

### Mode selection

Invocation: `/prd-to-automation <mode> [arguments]`, for example `/prd-to-automation govern docs/PRD.md`, `/prd-to-automation automate`, or `/prd-to-automation audit`. The mode and its arguments come from the request text. Without an explicit mode:

| Situation | Mode |
|---|---|
| PRD present, none of the four governance files exist | govern |
| Governance present and the user says create, bootstrap, generate, update, regenerate, merge | govern: run the governance audit first, then ask per file (overwrite, merge, skip) |
| All four governance files present, no `auto-develop.sh` | automate |
| `auto-develop.sh` present and the user says sync, adapt, align, update the script | automate (Sync) |
| User says check, review, compare, audit, drift, dry-run, validate | audit |
| Governance missing or incomplete and the user wants automation | govern first; automate never starts without all four files |

State the chosen mode in one line before doing any work. When a mode finds work that belongs to another mode, say so and switch modes; a switch that will write files needs the user's approval first. Never route to an external skill.

## The shared contract

`references/contract.md` defines once what govern writes, automate reads, and audit checks: the uncertainty markers, the priority levels, the memory rules, the AGENTS.md *Skill Policy* format and how it seeds `SKILL_MAP`, and the test-discipline fields (`TEST_POLICY`, `TEST_ELIGIBILITY`, `TARGETED_TEST_CMD` with `{TARGET}`), each with its optional or required status and the behaviour when it is absent. The contract carries its own version. Mode references link to it instead of repeating it.

## Boundaries

- **Critical**: govern writes only the four governance files and `memory/completed-phases.md`, only after the user's explicit per-file decision and approval. It never commits.
- **Critical**: automate writes only `MEMORY.md` (the *Current State* line and, when drift is open, one *Governance Drift* line, per its update rules), one entry in `memory/completed-phases.md`, and the generated artifacts. It never edits `SOUL.md`, `AGENTS.md`, or `CLAUDE.md`. A needed correction is `[GOVERNANCE DRIFT]` and goes through govern. It never starts without all four governance files. It never runs the real pipeline loop; validation means `bash -n`, `shellcheck`, `--dry-run`.
- **Critical**: audit writes nothing. Changes happen only after switching to govern or automate and after approval.
- **Critical**: generated scripts keep `bypassPermissions`, `danger-full-access`, and auto-merge off by default. They are reachable only via `--unattended` / `--auto-merge` behind the runtime `confirm_privileged_mode` gate, and are never written as defaults.
- **Required**: stack-agnostic. The generated script assumes no toolchain. Checks come verbatim from `CLAUDE.md` into `CHECKS=()`; an empty array is a valid no-op.
- **Required**: deterministic resolution. Skill routing and test eligibility resolve only from explicit `label:` / `title:` matchers, once per task, logged. No filesystem, registry, network, or semantic search. Never invent matchers.
- **Required**: models are chosen explicitly by the user per pipeline step in automate, never silently inherited from governance. A pick that differs from governance is generated as chosen and recorded as open `[GOVERNANCE DRIFT]` for govern.
- **Required**: the safe default `--permission-mode default` cannot answer permission prompts in headless `claude -p` runs. Every generated run guide tells the operator to pre-allow the implementer's tools (`.claude/settings.json` `permissions.allow` or `--allowedTools`); `--unattended` stays the explicit opt-in.
- **Required**: link, don't duplicate. Prompts and references instruct agents to read the governance files instead of copying large governance text.

## Presenting work

Every mode ends by presenting what was done or found, every decision that was not explicit in the inputs, and all open markers. govern and automate ask for explicit approval before writing anything; audit proposes which mode to switch to. No mode commits; the user reviews first.

## Backward compatibility

Projects whose governance came from `prd-to-governance` 1.2.0 and whose script came from `governance-to-automation` 1.2.2 keep working unchanged. An audit reports no drift merely because the skill changed. Scripts generated since 1.1.0 carry a header comment naming the skill and contract version they were generated with; a script without it predates 1.1.0.
