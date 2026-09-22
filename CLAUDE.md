# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Code skill-authoring repository**. It contains no application; its "source" is the **`governance-pipeline`** skill (Markdown plus one example shell script): a single skill with three modes that takes *other* projects from a PRD to governance files (govern), from governance to a generated `auto-develop.sh` (automate), and audits both for drift (audit).

```
Idea/PRD ─▶ govern ─▶ SOUL.md / AGENTS.md / CLAUDE.md / MEMORY.md ─▶ automate ─▶ auto-develop.sh
                       (the governance contract, references/contract.md)         (runs autonomously)
                                        ▲ audit reads all of it, writes nothing ▲
```

The skill merged `prd-to-governance` 1.2.0 (now govern) and `governance-to-automation` 1.2.2 (now automate) with **unchanged behaviour**. Those names appear in the skill only as provenance. `docs/PRD.md` (German) is the PRD this repository was built from; `docs/parity.md` proves every origin rule has a home.

## Skill anatomy

```
skills/governance-pipeline/
  SKILL.md              short entry point: project root, mode table, mode selection, boundaries (target under 150 lines, hard limit 200)
  references/
    contract.md         THE shared contract, versioned; markers, priorities, memory rules, Skill Policy, test discipline, field table
    govern.md           govern workflow (Steps 0-11), adaptation, quality checklist
    automate.md         automate workflow (Steps 0-7, Sync), adaptation, quality checklist
    audit.md            read-only: governance drift, script drift, validation, report format
    *-template.md       govern blueprints (soul, agents, claude, memory, completed-phases)
    auto-develop-template.md, prompt-builders.md, task-list-template.md, extraction-checklist.md
                        automate blueprints
  agents/openai.yaml    Codex metadata
```

A mode loads only the references it needs; `SKILL.md` names them per mode. Frontmatter uses only `name`, `description`, `license`, `metadata.version`; mode and arguments come from the request text, never from frontmatter.

## The contract (most important architecture)

`references/contract.md` is the single definition of what govern writes, automate reads, and audit checks. It has its own version. Mode references and blueprints **link to it and never restate it**. A change there is a contract change and must be labelled as such in `CHANGELOG.md`. Keep these invariants intact when editing anything:

- Review diffs **exclude** `MEMORY.md` (`git diff <base> -- . ':!MEMORY.md'`); the implement/fix/refactor steps write **one overwritten** "Next Up" line; only the post-review memory step writes completed work, to `memory/completed-phases.md`; **no-op fix detection** breaks the loop; `Depends on #N` hard-blocks; the checkpoint commit needs a **non-empty code diff**.
- **Boundary**: automate writes only `MEMORY.md` plus generated artifacts and never edits `SOUL.md` / `AGENTS.md` / `CLAUDE.md`; corrections go through govern. audit writes nothing. automate never starts without all four governance files and never runs the real loop.
- **Deterministic resolution**: skill routing and test eligibility come only from explicit `label:` / `title:` matchers, resolved once per task without filesystem, registry, network, or semantic search, and logged. Ambiguity injects nothing. `except` beats `include`; inert sets fail safe to `off`, never "test every task". The gate is a targeted red→green proof, not a no-regression gate; `required` may block, `preferred` never does. `{TARGET}` is allowlist-sanitised and runs via `bash -c`, never `eval`.
- **Safe defaults**: `bypassPermissions`, `danger-full-access`, and auto-merge are never defaults anywhere, including the fixtures. They sit behind `--unattended` / `--auto-merge` and the runtime `confirm_privileged_mode` gate (lists privileges, prompts `[y/N]`, skipped by `--dry-run` / `--yes`, refuses without a TTY); the tmux re-exec propagates the opt-in plus `--yes`.
- **Stack-agnostic**: checks come verbatim from the target project's `CLAUDE.md` into `CHECKS=()`; an empty array is a valid no-op. Do not reintroduce `package.json` guards or a default package manager.

## Authoring conventions

- Uncertainty markers: `[NEEDS PRD CLARIFICATION]`, `[NEEDS CODEBASE DISCOVERY]`, `[USER DECISION REQUIRED]`, `[GOVERNANCE DRIFT]`, `[NEEDS GOVERNANCE]`. Priority levels **Critical** / **Required** / **Advisory**, only where they sharpen real stakes. Both are defined in `contract.md`.
- **Link, don't duplicate**: prompts and references tell agents to *read* the governance files rather than copying them; mode references point to `contract.md` instead of repeating it.
- Cross-mode references are mode switches ("switch to govern mode"), never "install skill X". The origin skill names may appear only as provenance.
- The blueprints `auto-develop-template.md`, `prompt-builders.md`, `task-list-template.md`, `extraction-checklist.md` and the five governance templates are carried over verbatim from the origins; only cross-skill references were rewritten. Keep it that way unless the change is a deliberate, changelogged behaviour change.
- Skill text and README stay English. `docs/PRD.md` and internal notes may be German.
- Lowercase-hyphenated Markdown filenames, LF line endings (the repo sets `core.autocrlf=false`).

## Fixtures

`examples/auto-develop.payload-sample.sh` and `examples/refact-todo.md` are **sample outputs** generated for a Node/pnpm + Payload CMS project. They are not this repository's build system and are never run here. The sample script is a pre-refactor-pass snapshot and is intentionally not re-synced; `references/auto-develop-template.md` is the current pipeline shape. Both mirror the safe-by-default privileged policy.

## Commands

There is no build, lint, or test toolchain. The only checks:

```bash
bash -n examples/auto-develop.payload-sample.sh                      # syntax-check the fixture
shellcheck examples/auto-develop.payload-sample.sh                   # lint it (clean as of 1.0.0)
python <skill-creator>/scripts/quick_validate.py skills/governance-pipeline   # frontmatter validation
wc -l skills/governance-pipeline/SKILL.md                            # must stay at or below 200 lines
rg -n 'prd-to-governance|governance-to-automation' skills/           # only provenance mentions allowed
rg -n 'bypassPermissions|danger-full-access|merge --squash' skills/ examples/   # never as a default
```

The frontmatter validator is the `quick_validate.py` script shipped with Anthropic's `skill-creator` skill (allowed top-level keys: `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`). The origin changelogs did not record which tool validated them; this one enforces the same rule.

## Caveats

- The root `AGENTS.md` describes this repo's own contributor guidelines. It is *not* a governance file produced by the govern mode; do not treat it as pipeline input. There is no `SOUL.md` or `MEMORY.md` here, so do not add `@SOUL.md` / `@MEMORY.md` references.
- Publishing steps (GitHub repo creation, tags, the "superseded" banner in the two origin repos, archiving) are outward-facing and need the user's explicit go-ahead.
