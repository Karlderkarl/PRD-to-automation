# PRD-to-automation

A Claude Code skill that takes a project from idea to autonomous delivery in three modes:

| Mode | Input | Output |
|---|---|---|
| **govern** | a PRD and the current repository | `SOUL.md`, `AGENTS.md`, `CLAUDE.md`, `MEMORY.md`, `memory/completed-phases.md` |
| **automate** | those four governance files | a project-tailored, stack-agnostic `auto-develop.sh` (implement → check → dual review → fix → refactor → re-review → commit → PR), its task source, prompt builders, logging, and a run guide |
| **audit** | governance, repository, and script | one read-only report: governance drift, script drift, validation (`bash -n`, `shellcheck`, `--dry-run`) |

The two stages share one contract (`references/contract.md`): uncertainty markers, priority levels, the memory-management discipline, deterministic per-task skill routing, and an optional, opt-in test-discipline gate. Privileged execution (`bypassPermissions` / `danger-full-access`) and auto-merge are **off by default** in every generated script, reached only via explicit `--unattended` / `--auto-merge` flags behind a runtime confirmation prompt.

## Install

```bash
npx skills add Karlderkarl/PRD-to-automation
```

Requires [Claude Code](https://claude.ai/code).

## Usage

Claude Code auto-selects the skill when you ask it to create or refresh governance from a PRD, to generate or sync an auto-develop pipeline, or to audit either. You can also name the mode:

```
/prd-to-automation govern docs/PRD.md
/prd-to-automation automate
/prd-to-automation audit
```

Without a named mode, the skill picks one from the situation (PRD but no governance → govern; governance but no script → automate; "check", "drift", "dry-run" → audit) and states its choice in one line before working.

### A typical first run

1. **govern** reads the PRD, inspects the repository, interviews you on roles, git conventions, task source, and commands, then proposes the four files. Nothing is written before you approve per file.
2. **automate** extracts checks, roles, conventions, and memory rules from the governance, asks you to choose the model for every pipeline step, wires exactly one task source, and generates `auto-develop.sh`. It validates the script but never runs the real loop.
3. **Commit** the governance, the task source, and the script (the pipeline refuses to start on a dirty worktree).
4. **Dry run** it yourself: `./auto-develop.sh --dry-run` (or `/prd-to-automation audit`) confirms task selection without executing models. Then `./auto-develop.sh --max-issues 1`.

### Permissions for headless runs

The pipeline calls `claude -p`, which cannot answer permission prompts. Under the safe default (`--permission-mode default`) every tool the implementer needs must be allowed up front, otherwise the implement and fix steps change nothing and each task ends with "no code changes". Allow them in the target project's `.claude/settings.json`, for example:

```json
{
  "permissions": {
    "allow": ["Edit", "Write", "Bash(pytest:*)", "Bash(ruff:*)"]
  }
}
```

or pass `--allowedTools` in the implementer's runner line. Scope the `Bash(...)` entries to the project's own check commands. `--unattended` switches the implementer to `bypassPermissions` behind a confirmation prompt; it is the explicit opt-in for fully unattended runs, not the fix for a missing allowlist. Reviewers always run read-only, `--unattended` included.

A dependent task waits until its dependency is merged into the base branch, so a batch run without `--auto-merge` processes independent tasks only.

## Repository layout

```
skills/prd-to-automation/
  SKILL.md                         entry point: ground rules, mode selection, boundaries
  references/
    contract.md                    the shared contract (versioned)
    govern.md                      govern workflow and quality checklist
    automate.md                    automate workflow and quality checklist
    audit.md                       audit workflow and quality checklist
    soul-template.md, agents-template.md, claude-template.md,
    memory-template.md, completed-phases-template.md
                                   governance blueprints (govern)
    auto-develop-template.md, prompt-builders.md,
    task-list-template.md, extraction-checklist.md
                                   pipeline blueprints (automate)
  agents/openai.yaml               Codex metadata
examples/                          read-only fixtures: a generated script and a local task list
docs/PRD.md                        the PRD this repository was built from (German)
docs/parity.md                     every rule of the origin skills and where it lives now
```

`examples/` holds sample outputs generated for a Node/pnpm + Payload CMS project. They are fixtures, not this repository's build system, and are never executed here. The sample script predates the refactor pass and is intentionally not re-synced; the current pipeline shape is `references/auto-develop-template.md`.

## Relationship to pi-governance-pipeline

[`pi-governance-pipeline`](https://github.com/Karlderkarl/pi-governance-pipeline) is the sister project for the [pi](https://github.com/badlogic/pi-mono) harness. Same mode model (govern, automate, audit), different mechanics, on purpose:

| Aspect | PRD-to-automation (Claude Code) | pi-governance-pipeline (pi) |
|---|---|---|
| Pipeline | a Bash script generated per project | a versioned Node engine, wrapper in the project |
| Harness file | `CLAUDE.md` | `SYSTEM.md` and `.pi/APPEND_SYSTEM.md` |
| Contract | prose sections in `AGENTS.md` / `CLAUDE.md` | YAML block `pipeline-contract` in `AGENTS.md` |
| Review | Reviewer A/B, fix loop, refactor pass | three reviewers, controller, master, multi-provider |
| MEMORY.md | the implement step writes the "Next Up" line, the memory step writes the archive | only the engine writes, model roles never |

Neither replaces the other; pick the one for your harness.

## Origin

This skill merges two earlier Claude Code skills with unchanged behaviour, apart from the fixes to the inherited pipeline template listed in the CHANGELOG: [`prd-to-governance`](https://github.com/Karlderkarl/prd-to-governance) 1.2.0 became the govern mode and [`governance-to-automation`](https://github.com/Karlderkarl/governance-to-automation) 1.2.2 became the automate mode; their audit parts became the audit mode. Projects generated with those versions keep working unchanged. `docs/parity.md` maps every rule of both quality checklists to its new location. The PRD (`docs/PRD.md`) proposed the name `governance-pipeline`; repo and skill are named `PRD-to-automation` / `prd-to-automation` since 2026-09-22, leaving that name to the pi sister project.

## Security

Security issues should be reported through GitHub Private Vulnerability Reporting, not public issues. See `SECURITY.md`.

## License

MIT. See `LICENSE`.
