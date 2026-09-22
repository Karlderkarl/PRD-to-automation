# Repository Guidelines

## Project Structure & Module Organization
This is a Claude Code **skill-authoring** repository, not an application. It ships **one skill**, `prd-to-automation`, with three modes: govern (PRD to `SOUL.md` / `AGENTS.md` / `CLAUDE.md` / `MEMORY.md`), automate (governance to a generated, stack-agnostic `auto-develop.sh`), and audit (read-only drift report and validation).
- `skills/prd-to-automation/SKILL.md` is the short entry point (mode selection, boundaries); `references/` holds the shared contract (`contract.md`), the three mode workflows (`govern.md`, `automate.md`, `audit.md`), and the blueprints they load on demand.
- `examples/` holds **sample outputs** (`auto-develop.payload-sample.sh`, `refact-todo.md`) generated for a Node/pnpm + Payload CMS project: read-only fixtures, not this repository's build system. The sample script must not be run here.
- `docs/PRD.md` is the PRD this repo implements (German); `docs/parity.md` maps every origin rule to its new location. `CLAUDE.md` holds the architecture and the contract invariants; read it first.

## Build, Test, and Development Commands
There is no application toolchain (Markdown plus one example Bash script). The meaningful checks:
- `bash -n examples/auto-develop.payload-sample.sh` syntax-checks the fixture
- `shellcheck examples/auto-develop.payload-sample.sh` lints it
- `python <skill-creator>/scripts/quick_validate.py skills/prd-to-automation` validates the frontmatter
- `wc -l skills/prd-to-automation/SKILL.md` must stay at or below 200

When the automate mode generates a script, validate it the same way (`bash -n`, `shellcheck`, then `--dry-run`) before running. Generation must never execute the real loop.

## Coding Style & Naming Conventions
Match the style of the files you touch. All modes share the vocabulary defined in `references/contract.md`: uncertainty markers (`[NEEDS PRD CLARIFICATION]`, `[NEEDS CODEBASE DISCOVERY]`, `[USER DECISION REQUIRED]`, `[GOVERNANCE DRIFT]`, `[NEEDS GOVERNANCE]`), priority levels (**Critical** / **Required** / **Advisory**), "link, don't duplicate" (read governance, never copy large text; point to `references/contract.md` for the definitions, which is the authoritative wording wherever a rule is restated), and stack-agnostic generation (never assume a toolchain). Cross-mode references are mode switches, never external skills. Prefer lowercase-hyphenated Markdown filenames and LF line endings. Skill text and README are English; `docs/PRD.md` may be German.

## Testing Guidelines
No test framework. Validate changes by syntax- and lint-checking scripts, running the frontmatter validator, and dry-running generated pipelines. When changing the skill, keep the invariants in `references/contract.md` intact: memory discipline (diff exclusion of `MEMORY.md`, single overwritten "Next Up" line, archive-only completed work, no-op fix detection, `Depends on #N` blocking, non-empty checkpoint), deterministic skill and test resolution (explicit `label:` / `title:` matchers only, `except` beats `include`, inert sets fail safe to `off`, targeted red→green gate that is not a no-regression gate, `required` may block while `preferred` stays advisory, `{TARGET}` sanitised before `bash -c`), and safe defaults. A contract change is recorded as such in `CHANGELOG.md` and bumps the contract version. Update `docs/parity.md` whenever a checklist rule moves.

## Commit & Pull Request Guidelines
Use Conventional Commit prefixes (`feat:`, `fix:`, `docs:`, `chore:`). Keep commits scoped to one concern. Pull requests state the purpose, summarize the changed files, and note verification (`bash -n` / `shellcheck` / validator / `--dry-run`).

## Security & Configuration Tips
`.gitignore` excludes `.claude/settings.local.json` and packaged skill artifacts; never commit secrets, tokens, or machine-specific credentials. Privileged flags in generated scripts (`bypassPermissions`, `danger-full-access`, auto-merge) are **off by default** and require explicit user opt-in: scripts ship safe modes and reach privileged ones only via `--unattended` / `--auto-merge` behind a runtime `confirm_privileged_mode` prompt (skipped by `--dry-run` / `--yes`, refuses without a TTY), never hardcoded as defaults, not even in the fixtures. The automate mode may write only `MEMORY.md` plus generated artifacts; `SOUL.md` / `AGENTS.md` / `CLAUDE.md` are never edited by it, corrections go through the govern mode. The audit mode writes nothing.
