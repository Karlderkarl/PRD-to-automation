# auto-develop.sh Template

Structural blueprint for the generated pipeline. It is a proven issue-driven loop, generalized with `{{PLACEHOLDERS}}` the skill fills from governance. Keep the control flow; swap the project-specific values.

## Placeholder legend (governance → script)

| Placeholder | Source | Example |
|---|---|---|
| `{{IMPL_MODEL}}` / `{{REVIEW_A}}` / `{{REVIEW_B}}` | Step 3 confirmed model selection, seeded from AGENTS.md / CLAUDE.md | `opus` / `sonnet` / `codex` |
| `{{IMPL_LABEL}}` / `{{REVIEW_A_LABEL}}` / `{{REVIEW_B_LABEL}}` | Step 3 confirmed review plan | `Implementation (Opus)` / `Reviewer A (Sonnet)` / `Reviewer B (Codex)` |
| `{{IMPL_RUNNER}}` / `{{REVIEW_A_RUNNER}}` / `{{REVIEW_B_RUNNER}}` | Step 3 confirmed model/CLI mapping | the CLI binary per role, checked on `PATH` at startup: `claude` / `claude` / `codex` |
| `{{IMPL_RUNNER_CALL}}` / `{{REVIEW_A_RUNNER_CALL}}` / `{{REVIEW_B_RUNNER_CALL}}` | Step 3 confirmed model/CLI mapping | the runner body, prompt on stdin (`$1` is the prompt file). Implementer: `claude -p --model "$MODEL" --permission-mode "$CLAUDE_PERMISSION_MODE" --output-format text < "$1"` (codex: `codex exec -m "$MODEL" --sandbox "$CODEX_SANDBOX_MODE" - < "$1"`). Reviewer, always read-only with literal flags: `claude -p --model "$REVIEW_A_MODEL" --permission-mode default --disallowedTools "Edit Write NotebookEdit Bash" --output-format text < "$1"` (codex: `--sandbox read-only` plus `--output-last-message`). `return 1` for `{{REVIEW_B_RUNNER_CALL}}` when `REVIEW_B_ENABLED=false`. See *Reviewer runners* and *Headless permissions* under Generation rules |
| `{{REVIEW_B_ENABLED}}` | Step 3 confirmed review plan | `true` (dual A/B review) or `false` (single review). With `false`, fill `{{REVIEW_B}}` / `{{REVIEW_B_LABEL}}` / `{{REVIEW_B_RUNNER}}` empty and `{{REVIEW_B_RUNNER_CALL}}` with `return 1`; every Reviewer B code path stays in the script and is skipped at runtime |
| `{{PROJECT_NAME}}` | SOUL.md title / project name | `TaskTimer` (filled in the prompt-builder blueprint, `prompt-builders.md`) |
| `{{BASE_BRANCH}}` | AGENTS.md *Git Conventions* | `main` |
| `{{TASK_LABEL}}` | task source decision | `agent:auto` |
| `{{TASK_SOURCE_HAS_LABELS}}` | task source decision | `true` for GitHub Issues; `false` for a local task list or MEMORY.md "Next Up" (every `label:` matcher is then dead and warned) |
| `{{CHECK_CMDS[]}}` | CLAUDE.md *Development Commands* | stack-agnostic — whatever the project lists, e.g. `pnpm lint` / `uv run pytest` / `cargo test` / `make check` |
| `{{TOOLCHAIN_SETUP}}` | CLAUDE.md / SOUL.md stack | optional; only what governance specifies (PATH export, `corepack enable`, `source .venv/bin/activate`, …) — empty if none |
| `{{MEMORY_FILE}}` / `{{ARCHIVE_FILE}}` | MEMORY.md *Update Rules* | `MEMORY.md` / `memory/completed-phases.md` |
| `{{REFERENCE_DOCS}}` | SOUL.md *Reference Documents* | `setup-guide.md`; may be empty (filled in the prompt-builder blueprint) |
| `{{GOVERNANCE_REVIEW_FOCUS}}` | SOUL.md + AGENTS.md (see prompt-builders.md) | concise security/coding rule list (filled in the prompt-builder blueprint) |
| privileged execution | Step 3 user opt-in | **Not** a placeholder — generated scripts ship safe defaults (`default` / `workspace-write`) and reach privileged modes only via the runtime `--unattended` / `--auto-merge` flags behind `confirm_privileged_mode`. `--unattended` raises only the implementer's mode; reviewers are always read-only. Never hardcode `bypassPermissions` / `danger-full-access` as a default. |
| `{{TEST_POLICY}}` / `{{TEST_ELIGIBILITY[]}}` | AGENTS.md *Auto-Develop Policy* | `required` plus `label:backend=include`, `title:^docs:=except` |
| `{{TARGETED_TEST_CMD}}` | CLAUDE.md *Development Commands* | `pytest {TARGET}` / `pnpm test -- --runTestsByPath {TARGET}`; `{TARGET}` stands **unquoted** (the pipeline substitutes it shell-quoted) |
| `{{SKILL_MAP[]}}` | AGENTS.md *Skill Policy* + user-approved local entries (explicit matchers → skill) | `label:area:auth=security-hardening` — empty array only if both absent |

For the **local task-list** source (no GitHub Issues), replace the `gh`-based helpers, the reads at the top of `process_issue`, the PR/merge phase, and the candidate selection with the bash blueprint in `task-list-template.md` (branch, implement, check, review, commit, script-owned status flip, stop at the commit). Keep `--dry-run` **read-only** in this variant too: select/print the next task but do **not** flip its status or write the task file — a dirtied task file would trip the next run's clean-worktree guard. Skill resolution still runs once per task, but a task-list task has no labels: the generator sets `TASK_SOURCE_HAS_LABELS=false`, `resolve_skill` gets an empty label string, and only `title:` matchers (against the task title/body) can resolve a skill.
The same deterministic rule applies to **test eligibility** in the task-list variant: with `TASK_SOURCE_HAS_LABELS=false` every `label:` `TEST_ELIGIBILITY` matcher is **dead** and warned at runtime, so only `title:` matchers can make a task test-eligible. Dead matchers never arm the denylist base, so a `label:`-only `except` set resolves to inert → `off` (warned), never to "test every task". Flag it as `[GOVERNANCE DRIFT]` at generation/audit as well.

## Skeleton

```bash
#!/usr/bin/env bash
# Generated by prd-to-automation 1.2.0 (contract 1.2.0). Change it through /prd-to-automation automate (Sync).
set -euo pipefail
# Absolute path of this script, resolved before any cd: the tmux re-exec runs `bash <path>`,
# so a relative or bare $0 (`bash auto-develop.sh`, run from elsewhere) still resolves.
SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)" \
  || { echo "Cannot resolve the script directory." >&2; exit 1; }
SCRIPT_PATH="$SCRIPT_DIR/$(basename -- "${BASH_SOURCE[0]}")"

usage() {
  cat <<'EOF'
Usage: auto-develop.sh [OPTIONS]

Options:
  --max-issues <n>         Stop after completing n tasks/issues (default: 1)
  --issue <number>         Process only this specific issue
  --model <model>          Implementation/fix model
  --review-a <model>       Reviewer A model
  --review-a-effort <lvl>  Reviewer A reasoning effort (default: high)
  --review-b <model>       Reviewer B model (dual review only: rejected when
                           REVIEW_B_ENABLED=false)
  --review-b-effort <lvl>  Reviewer B reasoning effort (default: high; dual review only)
  --max-rounds <n>         Max review-fix rounds per issue (default: 100)
  --refactor               Run the post-review simplification pass (default)
  --no-refactor            Skip the post-review simplification pass
  --max-refactor-rounds <n> Max simplify->re-review rounds (default: 3)
  --unattended             Opt in to privileged unattended execution of the
                           IMPLEMENTER (claude: bypassPermissions; codex:
                           danger-full-access). Reviewers stay read-only. OFF by
                           default; prompts for confirmation (see --yes).
  --auto-merge             Squash-merge the PR after a clean review (OFF by default;
                           the run otherwise stops at the PR for human review).
  --yes                    Skip the privileged-mode confirmation prompt.
                           Required for non-interactive/detached (tmux) runs.
  --dry-run                Show planned work without executing model steps
  --tmux-session <name>    Launch the run in a detached tmux session, then exit
  --tmux-log <path>        Log file for detached tmux runs
  -h, --help               Show this help text

Examples:
  ./auto-develop.sh --max-issues 100
  ./auto-develop.sh --issue 42 --dry-run
  ./auto-develop.sh --max-issues 100 --tmux-session auto-develop
EOF
}

# Issue-driven development loop. Only processes issues labeled `{{TASK_LABEL}}`.
# Skips issues whose `Depends on #N` references are still open.
# Pipeline per issue:
#   implementer implements -> checks -> Reviewer A reviews -> Reviewer B reviews (skipped when
#   REVIEW_B_ENABLED=false) -> fix (if findings) -> re-check -> re-review -> repeat (correctness pass)
#   -> refactor (simplify to senior quality) -> re-check -> re-review -> repeat
#      until a refactor round changes nothing (refactor pass)
# Exits cleanly when no eligible issue remains.

# --- Defaults (override via flags) ---
MAX_ISSUES=1
TARGET_ISSUE=""
IMPL_LABEL="{{IMPL_LABEL}}"
MODEL="{{IMPL_MODEL}}"
IMPL_RUNNER="{{IMPL_RUNNER}}"
REVIEW_A_LABEL="{{REVIEW_A_LABEL}}"
REVIEW_A_MODEL="{{REVIEW_A}}"
REVIEW_A_RUNNER="{{REVIEW_A_RUNNER}}"
REVIEW_A_EFFORT="high"
REVIEW_B_ENABLED="{{REVIEW_B_ENABLED}}"  # true = dual A/B review | false = single review (Reviewer B never runs)
REVIEW_B_LABEL="{{REVIEW_B_LABEL}}"      # the REVIEW_B_* values stay defined (may be empty) when disabled
REVIEW_B_MODEL="{{REVIEW_B}}"
REVIEW_B_RUNNER="{{REVIEW_B_RUNNER}}"
REVIEW_B_EFFORT="high"
REVIEW_B_FLAG=false                      # set by --review-b*; rejected at startup when REVIEW_B_ENABLED=false
MAX_ROUNDS=100
REFACTOR=true                 # second pass: simplify to senior quality, re-reviewed like the correctness pass
MAX_REFACTOR_ROUNDS=3
DRY_RUN=false
TMUX_SESSION=""
TMUX_LOGFILE="logs/auto-develop.tmux.log"
NO_TMUX_REEXEC=false

LOGDIR="logs/issues"
BASE_BRANCH="{{BASE_BRANCH}}"
MEMORY_FILE="{{MEMORY_FILE}}"
ARCHIVE_FILE="{{ARCHIVE_FILE}}"
# Privileged execution is OFF by default. Do NOT hardcode bypassPermissions /
# danger-full-access as defaults — that is exactly what static scanners flag and
# what `automate.md` Step 3 forbids without explicit opt-in. --unattended raises the
# IMPLEMENTER's mode only (the variable matching IMPL_RUNNER); reviewer runners never read
# these two variables and always run read-only. --auto-merge enables the squash-merge. Nothing
# runs unattended unless the operator opts in AND confirms (confirm_privileged_mode).
CLAUDE_PERMISSION_MODE="default"               # claude implementer only; --unattended -> bypassPermissions
CODEX_SANDBOX_MODE="workspace-write"           # codex implementer only; --unattended -> danger-full-access
UNATTENDED=false
AUTO_MERGE=false
ASSUME_YES=false                               # --yes: skip the interactive confirmation (required for detached/tmux)
TEST_POLICY="{{TEST_POLICY}}"                  # off | preferred | required (opt-in; absent/empty => off)
TARGETED_TEST_CMD="{{TARGETED_TEST_CMD}}"      # must contain {TARGET}, unquoted (substituted shell-quoted); may be empty when policy is off
TASK_SOURCE_HAS_LABELS="{{TASK_SOURCE_HAS_LABELS}}"  # true (GitHub Issues) | false (task list, MEMORY.md Next Up): label: matchers are then dead
TARGETED_TEST_FILE=""                          # set per issue inside process_issue
FROZEN_TARGETED_TEST_TARGET=""                 # set once a RED target is validated; reused for all later GREEN reruns
TEST_GATE_ACTIVE=false                         # per issue: true only when the HARD green gate is armed (required + confirmed RED)

# Skill resolution: explicit matchers -> skill, seeded from AGENTS.md "Skill Policy".
# Each entry is "<type>:<pattern>=<skill>" (whitespace around ':' and '=' is
# tolerated, so the spaced "<type>:<pattern> = <skill>" form the extraction
# checklist documents parses identically):
#   label:<issue-label>   match a task label  (most explicit; GitHub-issue source)
#   title:<ere>           match the task title/body (extended regex)
# A title: regex is tested against ONE string, "<title><newline><body>": `^` anchors the title
# start, `$` the END OF THE BODY, and `.` also matches the newline. To match the title alone,
# anchor its prefix (`^docs:`) and do not end the pattern with `$`.
# Determinism rules (see resolve_skill): 0 distinct matches -> "(none)";
# exactly 1 distinct -> chosen; >1 distinct -> "(ambiguous)" and NOTHING is injected.
# No registry, no network, no semantic guessing. Empty array is a valid no-op default;
# an operator may also author entries locally (e.g. from a skill's own tags/triggers).
SKILL_MAP=(
  {{FOR each entry in SKILL_MAP}} "{{entry}}"   # e.g. "label:area:auth=security-hardening"
  {{END}}
)

# Test policy: explicit matchers -> include/except, seeded from AGENTS.md "Auto-Develop
# Policy". Entirely absent is the valid no-op default (off). Deterministic resolution
# (see resolve_test_policy) — `except` always wins over `include`, and the base default
# for an unmatched task depends on whether any include matchers are DECLARED:
#   task matches an except       -> exempt (off)                 [explicit deny beats allow]
#   else task matches an include -> eligible
#   else, ANY include declared   -> not eligible (allowlist base; a dead/typo include still counts)
#   else (only except declared, >=1 usable) -> eligible (denylist base: "test all except …")
# Dead matchers (unknown type/effect, invalid regex, empty label pattern, any label: on a
# label-less source) are warned and never arm the denylist base. title: regexes see
# "<title><newline><body>" exactly as in SKILL_MAP above (`$` anchors the body end).
# This makes "test everything except X" expressible with a single except matcher, and
# is fully deterministic (no "ambiguous" outcome, no heuristic). `required` without
# TARGETED_TEST_CMD is not silently enforced: the script degrades it to `preferred` and
# logs [GOVERNANCE DRIFT]. A set policy with an EMPTY eligibility array is inert -> off
# (and warned: governance is likely incomplete; should be caught at generation/audit).
TEST_ELIGIBILITY=(
  {{FOR each entry in TEST_ELIGIBILITY}} "{{entry}}"   # e.g. "label:backend=include"
  {{END}}
)

# Validation commands, taken verbatim from CLAUDE.md (stack-agnostic — no assumption
# about Node/Python/etc). Each runs in order; any non-zero exit fails the check phase.
CHECKS=(
  {{FOR each cmd in CHECK_CMDS}} "{{cmd}}"   # e.g. "pnpm lint", "uv run pytest", "cargo test", "go vet ./..."
  {{END}}
)
{{TOOLCHAIN_SETUP}}                            # optional; empty unless governance specifies setup

# --- Arg parsing (keep this block; it backs the advertised CLI contract). The --review-b* cases
#     stay in a single-review script too; startup rejects them there (REVIEW_B_ENABLED=false). ---
# A flag that takes a value dies with the usage text when the value is missing (instead of
# an unbound-variable error under `set -u`, or silently eating the next flag).
need_val() { [[ $# -ge 2 && -n "$2" && "$2" != -* ]] || { echo "Option $1 needs a value." >&2; usage >&2; exit 1; }; }
while [[ $# -gt 0 ]]; do
  case "$1" in
    --max-issues)      need_val "$@"; MAX_ISSUES="$2";       shift 2 ;;
    --issue)           need_val "$@"; TARGET_ISSUE="$2";     shift 2 ;;
    --model)           need_val "$@"; MODEL="$2";            shift 2 ;;
    --review-a)        need_val "$@"; REVIEW_A_MODEL="$2";   shift 2 ;;
    --review-a-effort) need_val "$@"; REVIEW_A_EFFORT="$2";  shift 2 ;;
    --review-b)        need_val "$@"; REVIEW_B_MODEL="$2";  REVIEW_B_FLAG=true; shift 2 ;;
    --review-b-effort) need_val "$@"; REVIEW_B_EFFORT="$2"; REVIEW_B_FLAG=true; shift 2 ;;
    --max-rounds)      need_val "$@"; MAX_ROUNDS="$2";       shift 2 ;;
    --refactor)        REFACTOR=true;         shift ;;
    --no-refactor)     REFACTOR=false;        shift ;;
    --max-refactor-rounds) need_val "$@"; MAX_REFACTOR_ROUNDS="$2"; shift 2 ;;
    --unattended)      UNATTENDED=true;       shift ;;
    --auto-merge)      AUTO_MERGE=true;       shift ;;
    --yes)             ASSUME_YES=true;       shift ;;
    --dry-run)         DRY_RUN=true;          shift ;;
    --tmux-session)    need_val "$@"; TMUX_SESSION="$2";     shift 2 ;;
    --tmux-log)        need_val "$@"; TMUX_LOGFILE="$2";     shift 2 ;;
    --no-tmux-reexec)  NO_TMUX_REEXEC=true;   shift ;;
    -h|--help)         usage; exit 0 ;;
    *)                 echo "Unknown option: $1" >&2; usage; exit 1 ;;
  esac
done

log()  { echo "[auto-develop $(date +%H:%M:%S)] $*"; }
die()  { log "FATAL: $*" >&2; exit 1; }

# Raise the IMPLEMENTER's privileged value only when the operator explicitly opted in. Reviewer
# runners carry literal read-only flags and never read these variables (see Model runners).
if [[ "$UNATTENDED" == true ]]; then
  case "$IMPL_RUNNER" in
    claude) CLAUDE_PERMISSION_MODE="bypassPermissions" ;;
    codex)  CODEX_SANDBOX_MODE="danger-full-access" ;;
    *)      die "--unattended: no privileged mode is wired for implementer runner '$IMPL_RUNNER'." ;;
  esac
fi

# Run from the repository root: code_diff's `-- .` pathspec, the clean-worktree guard and the
# relative LOGDIR/MEMORY_FILE paths are all root-relative; from a subdirectory they would see
# only part of the tree.
REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null)" || die "Not inside a git work tree; run from the target repository."
cd "$REPO_ROOT" || die "Cannot cd to $REPO_ROOT"
[[ "$MAX_ISSUES" =~ ^[0-9]+$ && "$MAX_ROUNDS" =~ ^[0-9]+$ && "$MAX_REFACTOR_ROUNDS" =~ ^[0-9]+$ ]] \
  || die "--max-issues, --max-rounds and --max-refactor-rounds take a non-negative integer."
[[ -z "$TARGET_ISSUE" || "$TARGET_ISSUE" =~ ^[0-9]+$ ]] || die "--issue takes an issue number."
case "$TASK_SOURCE_HAS_LABELS" in true|false) ;; *) die "TASK_SOURCE_HAS_LABELS must be true or false." ;; esac
case "$REVIEW_B_ENABLED" in true|false) ;; *) die "REVIEW_B_ENABLED must be true or false." ;; esac
# Single review: the --review-b* flags are rejected, not silently ignored (Reviewer B never runs).
[[ "$REVIEW_B_ENABLED" == true || "$REVIEW_B_FLAG" == false ]] \
  || die "--review-b/--review-b-effort need REVIEW_B_ENABLED=true; this pipeline is single-review (add Reviewer B via automate Sync)."
[[ "$REVIEW_B_ENABLED" == false || -n "$REVIEW_B_RUNNER" ]] || die "REVIEW_B_ENABLED=true needs REVIEW_B_RUNNER."
REVIEW_PLAN="$REVIEW_A_LABEL"; RUNNERS=("$IMPL_RUNNER" "$REVIEW_A_RUNNER")
if [[ "$REVIEW_B_ENABLED" == true ]]; then REVIEW_PLAN+=", $REVIEW_B_LABEL"; RUNNERS+=("$REVIEW_B_RUNNER"); fi
# Fail fast when a selected model CLI is missing (every runner step would crash otherwise).
# A dry run runs no model, so it only warns.
for cli in "${RUNNERS[@]}"; do
  [[ -z "$cli" ]] || command -v "$cli" >/dev/null 2>&1 && continue
  [[ "$DRY_RUN" == true ]] || die "Runner CLI '$cli' not found on PATH."
  log "[dry-run] WARN: runner CLI '$cli' not found on PATH."
done

# Gate every privileged opt-in behind an explicit confirmation. A dry run touches
# nothing, so it is exempt. With --yes the prompt is skipped; without a TTY and
# without --yes we refuse rather than block a detached run forever.
confirm_privileged_mode() {
  [[ "$DRY_RUN" == true ]] && return 0
  local -a privileged=()
  [[ "$CLAUDE_PERMISSION_MODE" == "bypassPermissions" ]] && \
    privileged+=("the implementer (claude) runs with --permission-mode bypassPermissions (no per-action approval); reviewers stay read-only")
  [[ "$CODEX_SANDBOX_MODE" == "danger-full-access" ]] && \
    privileged+=("the implementer (codex) runs with --sandbox danger-full-access; reviewers stay read-only")
  [[ "$AUTO_MERGE" == true ]] && \
    privileged+=("PRs are squash-merged automatically after a clean review")
  [[ ${#privileged[@]} -eq 0 ]] && return 0   # fully safe defaults; nothing to confirm
  log "PRIVILEGED UNATTENDED MODE requested:"
  local item; for item in "${privileged[@]}"; do log "  - $item"; done
  if [[ "$ASSUME_YES" == true ]]; then log "Confirmed via --yes."; return 0; fi
  [[ -t 0 ]] || die "Privileged mode needs confirmation but no TTY is attached. Re-run with --yes."
  local reply=""; read -r -p "Proceed in privileged unattended mode? [y/N] " reply || true  # EOF -> empty -> clean abort
  [[ "$reply" =~ ^[Yy]$ ]] || die "Aborted by operator."
}
# Strip leading/trailing whitespace (used to parse SKILL_MAP tolerantly).
trim() { local s="$1"; s="${s#"${s%%[![:space:]]*}"}"; printf '%s' "${s%"${s##*[![:space:]]}"}"; }

EFFECTIVE_TEST_POLICY="$(trim "$TEST_POLICY")"
# Absent/empty (unfilled placeholder) is the valid backward-compatible default.
[[ -z "$EFFECTIVE_TEST_POLICY" ]] && EFFECTIVE_TEST_POLICY="off"
case "$EFFECTIVE_TEST_POLICY" in
  off|preferred|required) ;;
  *) log "WARN: unknown TEST_POLICY '$EFFECTIVE_TEST_POLICY'; treating as off."; EFFECTIVE_TEST_POLICY="off" ;;
esac
if [[ "$EFFECTIVE_TEST_POLICY" == "required" && -z "$TARGETED_TEST_CMD" ]]; then
  EFFECTIVE_TEST_POLICY="preferred"
  log "[GOVERNANCE DRIFT] AGENTS.md sets TEST_POLICY=required but CLAUDE.md has no TARGETED_TEST_CMD; degrading enforcement to preferred."
fi
if [[ -n "$TARGETED_TEST_CMD" && "$TARGETED_TEST_CMD" != *"{TARGET}"* ]]; then
  die "TARGETED_TEST_CMD must contain a literal {TARGET} token"
fi
# The pipeline substitutes {TARGET} already shell-quoted; quotes around it would be literal.
if [[ "$TARGETED_TEST_CMD" == *[\"\']\{TARGET\}* || "$TARGETED_TEST_CMD" == *\{TARGET\}[\"\']* ]]; then
  die "TARGETED_TEST_CMD must use {TARGET} unquoted (the pipeline quotes it)"
fi

build_reexec_args() {
  local -a args
  args+=(--max-issues "$MAX_ISSUES")
  [[ -n "$TARGET_ISSUE" ]] && args+=(--issue "$TARGET_ISSUE")
  args+=(--model "$MODEL")
  args+=(--review-a "$REVIEW_A_MODEL" --review-a-effort "$REVIEW_A_EFFORT")
  # Single review: no --review-b* (the child would reject them).
  [[ "$REVIEW_B_ENABLED" == true ]] && args+=(--review-b "$REVIEW_B_MODEL" --review-b-effort "$REVIEW_B_EFFORT")
  args+=(--max-rounds "$MAX_ROUNDS")
  [[ "$REFACTOR" == false ]] && args+=(--no-refactor)
  args+=(--max-refactor-rounds "$MAX_REFACTOR_ROUNDS")
  [[ "$UNATTENDED" == true ]] && args+=(--unattended)
  [[ "$AUTO_MERGE" == true ]] && args+=(--auto-merge)
  [[ "$DRY_RUN" == true ]] && args+=(--dry-run)
  # The human already confirmed in the foreground; the detached run has no TTY.
  args+=(--yes)
  args+=(--no-tmux-reexec)
  printf '%q ' "${args[@]}"
}

launch_in_tmux_if_requested() {
  [[ -z "$TMUX_SESSION" ]] && return 0
  [[ "$NO_TMUX_REEXEC" == true ]] && return 0
  [[ -n "${TMUX:-}" ]] && { log "Already inside tmux session '$TMUX_SESSION'; continuing in foreground."; return 0; }
  command -v tmux >/dev/null 2>&1 || die "tmux is required for --tmux-session"
  tmux has-session -t "$TMUX_SESSION" 2>/dev/null && die "tmux session '$TMUX_SESSION' already exists"
  mkdir -p "$(dirname "$TMUX_LOGFILE")"
  local reexec_args
  reexec_args="$(build_reexec_args)"
  # `bash <absolute path>`: a bare/relative $0 is not a command inside the new session.
  tmux new-session -d -s "$TMUX_SESSION" "cd $(printf '%q' "$PWD") && bash $(printf '%q' "$SCRIPT_PATH") $reexec_args | tee -a $(printf '%q' "$TMUX_LOGFILE")"
  log "Detached tmux session '$TMUX_SESSION' started."
  log "Reattach with: tmux attach -t $TMUX_SESSION"
  log "Live log: $TMUX_LOGFILE"
  exit 0
}

# Confirm before detaching, so the human approves in the foreground and the
# detached tmux child inherits the approval via the propagated --yes.
confirm_privileged_mode
launch_in_tmux_if_requested

# --- Worktree guards: never mix pre-existing changes into the run ---
git_status_outside_logs() { git status --porcelain --untracked-files=all -- . ":(exclude)$LOGDIR"; }
has_repo_changes_outside_logs() { [[ -n "$(git_status_outside_logs)" ]]; }
require_clean_worktree() {
  if has_repo_changes_outside_logs; then
    log "Working tree dirty outside $LOGDIR. Refusing to mix changes."; return 1; fi; }
# Stage all work, then unstage the log dir. NOTE: do NOT use
# `git add -A -- . ":(exclude)$LOGDIR"` — when $LOGDIR is gitignored that pathspec
# makes `git add` exit 1 (matched-but-ignored path), which aborts the run under
# `set -e`. Plain `git add -A` skips ignored paths silently (exit 0); the reset then
# drops logs if they happen to be tracked.
stage_repo_changes() { git add -A; git reset -q -- "$LOGDIR" >/dev/null 2>&1 || true; }

# Abort helper for the per-issue failure paths: DISCARD any in-progress work, then return
# to BASE_BRANCH. A bare `git checkout "$orig"` is unsafe — half-written impl/fix changes
# either block the checkout or get carried onto the base branch, tripping the next run's
# clean-worktree guard. Before the correctness checkpoint commit HEAD is still at the base
# tip, so `reset --hard` only drops uncommitted work; callers after the checkpoint (failed
# memory step/amend/push/PR) keep the committed issue branch and only drop leftovers. `clean` excludes
# $LOGDIR explicitly so this issue's freshly written FAILURE logs survive for debugging even
# if the operator never gitignored logs/ (don't rely on the gitignore for that).
return_to_base() {
  git reset --hard HEAD >/dev/null 2>&1 || true
  git clean -fd -e "$LOGDIR" >/dev/null 2>&1 || true
  git checkout "$BASE_BRANCH" >/dev/null 2>&1 || log "WARN: could not return to $BASE_BRANCH"; }

# Fast-forward BASE_BRANCH to its upstream before each task branches. M5 sees a dependency as
# CLOSED on GitHub once its PR is merged there; without this its code would be missing locally
# (no --auto-merge -> no `git pull`). No upstream -> no-op. ff-only, never forced: a diverged
# base fails the task with a clear log. Needs a clean tree (callers check first).
refresh_base() {
  local remote
  git checkout -q "$BASE_BRANCH" || { log "ERROR: cannot check out $BASE_BRANCH."; return 1; }
  remote="$(git config --get "branch.$BASE_BRANCH.remote" || true)"
  [[ -n "$remote" ]] || { log "No upstream for $BASE_BRANCH; using the local base as-is."; return 0; }
  git fetch -q "$remote" \
    || { log "ERROR: git fetch $remote failed; cannot refresh $BASE_BRANCH."; return 1; }
  git merge -q --ff-only "$BASE_BRANCH@{upstream}" \
    || { log "ERROR: $BASE_BRANCH cannot fast-forward to its upstream (diverged?); resolve by hand, never forced."; return 1; }; }

# M7: governance is read-only at runtime. Checked before the checkpoint commit and before the
# final amend (and before a refactor round is folded): any change to SOUL.md / AGENTS.md /
# CLAUDE.md against the base, tracked or untracked, fails the step.
governance_untouched() {   # fails closed: a git error counts as "changed"
  local untracked
  git diff --quiet "$BASE_BRANCH" -- SOUL.md AGENTS.md CLAUDE.md || return 1
  untracked="$(git ls-files --others --exclude-standard -- SOUL.md AGENTS.md CLAUDE.md)" || return 1
  [[ -z "$untracked" ]]; }

# --- Model runners: generate these from the Step 3 confirmed model/CLI mapping.
# Example only: Sonnet/Opus may route through `claude -p`, Codex through `codex exec`.
# Do not assume those defaults; write the runner functions to match the user's selection.
# Only the IMPLEMENTER runner reads $CLAUDE_PERMISSION_MODE / $CODEX_SANDBOX_MODE (raised by
# --unattended). REVIEWER runners are read-only regardless of --unattended and use literal flags:
# claude `--permission-mode default --disallowedTools "Edit Write NotebookEdit Bash"` (the diff is
# in the prompt; read-only tools stay available), codex `--sandbox read-only`.
# REVIEWER runners must emit ONLY the model's final message on stdout and must NOT `2>&1`:
# run_review scans stdout for the verdict and keeps stderr in a sidecar. `claude -p
# --output-format text` already prints just the final message; `codex exec` prints a header,
# the echoed prompt and reasoning summaries, so use `--output-last-message <file>` and `cat`
# that file (or `--json` plus jq) — otherwise any numbered line in that noise reads as a finding.
run_impl_model() {      # <prompt_file>
  {{IMPL_RUNNER_CALL}}
}
run_review_a_model() {  # <prompt_file>
  {{REVIEW_A_RUNNER_CALL}}
}
# Never called when REVIEW_B_ENABLED=false (the generator then fills the body with `return 1`):
run_review_b_model() {  # <prompt_file>
  {{REVIEW_B_RUNNER_CALL}}
}

slugify() { echo "$1" | tr '[:upper:]' '[:lower:]' \
  | sed 's/[^a-z0-9]/-/g; s/--*/-/g; s/^-//; s/-$//' | cut -c1-40; }
ensure_logdir() { local d="$LOGDIR/$1"; mkdir -p "$d"; echo "$d"; }

# --- Issue eligibility + dependency blocking ---
issue_has_label() {  # <n> -> non-zero when the label is missing OR gh failed
  local n
  n="$(gh issue view "$1" --json labels \
    --jq '[.labels[].name]|map(select(.=="{{TASK_LABEL}}"))|length')" || return 1
  [[ "$n" -gt 0 ]]; }
check_dependencies() {  # <n> -> non-zero if any "Depends on #N" is still open OR unverifiable
  local body lines deps d state
  # Fail CLOSED: the call site runs this inside `|| { ...; continue; }`, where bash suspends
  # `set -e`, so a failed `gh` call must be checked explicitly. Otherwise an unreadable body
  # silently means "no dependencies" and the Depends-on gate is bypassed.
  body="$(gh issue view "$1" --json body --jq '.body')"     || { log "Cannot read #$1 (gh failed); treating it as blocked."; return 1; }
  # Every #N on a "Depends on" line counts ("Depends on #12, #13", "Depends on: #12") — contract M5.
  lines="$(grep -oiE 'depends on[^[:cntrl:]]*' <<< "$body" || true)"
  # A cross-repository reference (owner/repo#13, or an issue/PR URL) is NOT the local #13: fail
  # closed instead of checking the wrong issue (or none).
  if grep -qE '[[:alnum:]_.-]#[0-9]+|/(issues|pull)/[0-9]+' <<< "$lines"; then
    log "#$1: cross-repository dependency not supported; treating as blocked."; return 1; fi
  deps="$(grep -oE '#[0-9]+' <<< "$lines" | tr -d '#' || true)"
  for d in $deps; do
    state="$(gh issue view "$d" --json state --jq '.state')"     || { log "Cannot read #$d (gh failed); treating it as blocked."; return 1; }
    [[ "$state" != "CLOSED" ]] && { log "Blocked by #$d"; return 1; }; done; return 0; }

# Note: without --auto-merge a dependency's issue stays OPEN until a human merges its PR, so a
# batch run processes independent issues only; dependents wait for a later run.

# Finished work awaiting review is never reprocessed: a branch with own commits ahead of the base
# (an earlier run got past the checkpoint: memory step, amend, push or PR failed, or the PR is
# open) or (GitHub variant) an open PR for the branch means "awaiting review / finish manually".
# Reprocessing would stack a second run on top, and under TEST_POLICY=required the test-first
# phase would fail forever with NOT RED. Returns 0 awaiting (AWAIT_REASON set), 1 free,
# 2 cannot tell (git/gh failed; the caller fails the issue).
branch_awaiting_review() {  # <branch>
  local b="$1" own prs
  AWAIT_REASON=""
  if git show-ref --verify --quiet "refs/heads/$b"; then
    own="$(git rev-list --count "$BASE_BRANCH..$b")" || return 2
    [[ "$own" -gt 0 ]] && { AWAIT_REASON="branch $b has $own own commit(s) ahead of $BASE_BRANCH"; return 0; }
  fi
  # GitHub variant only (drop for a local task list):
  prs="$(gh pr list --head "$b" --state open --json number --jq 'length')" || return 2
  [[ "$prs" -gt 0 ]] && { AWAIT_REASON="an open PR exists for $b"; return 0; }
  return 1; }

# Branch the issue from BASE_BRANCH (never from the current HEAD) so successive issues in a
# --max-issues > 1 run never stack on an earlier, still-unmerged issue branch — otherwise
# issue N's review diff would include issue N-1's code. Takes the branch NAME and only checks
# out (no echo): a `$(...)` wrapper would swallow a failed checkout and the run would go on
# committing to the base branch. A leftover branch with no own commits (earlier failure before
# the checkpoint) is recreated from the CURRENT base tip. A branch WITH own commits is skipped
# by the caller (branch_awaiting_review); this refuses to reset one rather than lose commits.
create_issue_branch() {  # <branch>
  local b="$1"
  if git show-ref --verify --quiet "refs/heads/$b" && ! git merge-base --is-ancestor "$b" "$BASE_BRANCH"; then
    log "ERROR: $b has own commits; refusing to reset it."; return 1; fi
  git checkout -B "$b" "$BASE_BRANCH"; }

# --- Checks: run each command in CHECKS[] in order; auto-fix once on failure.
#     Stack-agnostic: no package.json/runtime guard — the commands ARE the toolchain.
#     If CHECKS is empty (project declares none), the phase is a no-op pass. ---
run_checks() {  # <logfile>
  local failed=false cmd
  [[ ${#CHECKS[@]} -eq 0 ]] && { echo "SKIP: no checks declared in CLAUDE.md" > "$1"; return 0; }
  {
    for cmd in "${CHECKS[@]}"; do
      echo "=== $cmd ==="
      bash -c "$cmd" 2>&1 || failed=true   # bash -c (not eval): run the governance command string in a subshell
    done
  } > "$1"
  [[ "$failed" == false ]]; }
resolve_test_policy() {  # <labels> <title> <body> <logdir>
  local labels="$1" title="$2" body="$3" logdir="$4"
  local text="$title"$'\n'"$body"
  local -a reasons=() warnings=()
  local entry lhs type pat effect matched type_ok regex_rc l include_count=0 except_count=0 decl_include=0 decl_except=0
  RESOLVED_TEST_POLICY="off"
  RESOLVED_TEST_REASON="test policy disabled"
  if [[ "$EFFECTIVE_TEST_POLICY" == "off" ]]; then
    {
      echo "searched: labels=[$(printf '%s' "$labels" | tr '\n' ',')] title=[$title]"
      echo "matches:   (none)"; echo "chosen:    off"; echo "reason:    $RESOLVED_TEST_REASON"
    } > "$logdir/test-policy.log"
    return 0
  fi
  # Policy is set but no eligibility matchers exist -> inert. Safe (off), but flag it:
  # this is governance incompleteness ([NEEDS GOVERNANCE]) that generation/audit should
  # have caught. Do not silently pretend testing is simply disabled.
  if [[ ${#TEST_ELIGIBILITY[@]} -eq 0 ]]; then
    RESOLVED_TEST_REASON="policy=$EFFECTIVE_TEST_POLICY but TEST_ELIGIBILITY is empty (inert) -> off"
    {
      echo "searched: labels=[$(printf '%s' "$labels" | tr '\n' ',')] title=[$title]"
      echo "matches:   (none)"; echo "chosen:    off"; echo "reason:    $RESOLVED_TEST_REASON"
    } > "$logdir/test-policy.log"
    log "WARN: TEST_POLICY=$EFFECTIVE_TEST_POLICY but no TEST_ELIGIBILITY matchers — policy is inert (treat as [NEEDS GOVERNANCE]). See $logdir/test-policy.log"
    return 0
  fi
  for entry in "${TEST_ELIGIBILITY[@]}"; do
    [[ "$entry" == *=* ]] || { warnings+=("malformed TEST_ELIGIBILITY entry '$entry' (no '=include'/'=except')"); continue; }
    effect="$(trim "${entry##*=}")"; lhs="${entry%=*}"
    type="$(trim "${lhs%%:*}")"; pat="$(trim "${lhs#*:}")"
    matched=false; type_ok=false
    case "$type" in
      label) if [[ "$TASK_SOURCE_HAS_LABELS" != true ]]; then
               warnings+=("dead label: matcher '$entry': this task source has no labels [GOVERNANCE DRIFT]")
             elif [[ -z "$pat" ]]; then
               warnings+=("dead label: matcher '$entry': empty pattern")
             else
               type_ok=true; while IFS= read -r l; do [[ -n "$l" && "$l" == "$pat" ]] && matched=true; done <<< "$labels"
             fi ;;
      title) if [[ "$text" =~ $pat ]] 2>/dev/null; then
               type_ok=true; matched=true
             else
               # shellcheck disable=SC2319  # intended: $? of the [[ =~ ]] test (2 = invalid regex)
               regex_rc=$?
               if [[ "$regex_rc" -eq 2 ]]; then
                 warnings+=("invalid title: regex '$pat' (entry '$entry') — treated as no match")
               else
                 type_ok=true
               fi
             fi ;;
      *)     warnings+=("unknown TEST_ELIGIBILITY matcher type '$type' in '$entry'") ;;
    esac
    # Base default: ANY declared include — even a dead/typo one — keeps the allowlist base, so a
    # typo can never flip the set to a denylist. Only a WELL-FORMED except (usable type, valid
    # regex) counts toward the denylist base ("test all except …"); a dead one must never arm it.
    case "$effect" in
      include) decl_include=$((decl_include + 1)) ;;                       # declared allow intent
      except)  if [[ "$type_ok" == true ]]; then decl_except=$((decl_except + 1)); fi ;;   # usable deny intent
      *)       warnings+=("unknown TEST_ELIGIBILITY effect '$effect' in '$entry'") ;;
    esac
    [[ "$matched" == true ]] || continue
    reasons+=("$type:$pat -> $effect")
    case "$effect" in
      include) include_count=$((include_count + 1)) ;;
      except)  except_count=$((except_count + 1)) ;;
    esac
  done
  # Deterministic precedence: except wins; then include; then base default by DECLARED intent.
  # Any declared include -> allowlist base. The denylist base ("eligible unless excepted") fires
  # ONLY when no include is declared and a usable except matcher was; otherwise the set is inert
  # -> fail safe to off (governance incompleteness, [NEEDS GOVERNANCE]), never "test every task".
  if [[ "$except_count" -gt 0 ]]; then
    RESOLVED_TEST_POLICY="off"
    RESOLVED_TEST_REASON="exempt (except wins) via $(IFS='; '; echo "${reasons[*]}")"
  elif [[ "$include_count" -gt 0 ]]; then
    RESOLVED_TEST_POLICY="$EFFECTIVE_TEST_POLICY"
    RESOLVED_TEST_REASON="eligible via $(IFS='; '; echo "${reasons[*]}")"
  elif [[ "$decl_include" -gt 0 ]]; then
    RESOLVED_TEST_POLICY="off"
    RESOLVED_TEST_REASON="not test-eligible (allowlist base: include matchers declared, none matched)"
  elif [[ "$decl_except" -gt 0 ]]; then
    RESOLVED_TEST_POLICY="$EFFECTIVE_TEST_POLICY"
    RESOLVED_TEST_REASON="eligible (denylist base: only except matchers declared, none matched)"
  else
    RESOLVED_TEST_POLICY="off"
    RESOLVED_TEST_REASON="inert: no usable include/except matcher (all malformed/unknown/invalid/dead) -> off"
    warnings+=("TEST_ELIGIBILITY has no usable matcher — policy inert (treat as [NEEDS GOVERNANCE])")
  fi
  {
    echo "searched: labels=[$(printf '%s' "$labels" | tr '\n' ',')] title=[$title]"
    echo "matches:"; printf '  - %s\n' "${reasons[@]:-(none)}"
    [[ ${#warnings[@]} -gt 0 ]] && { echo "warnings:"; printf '  - %s\n' "${warnings[@]}"; }
    echo "chosen:    $RESOLVED_TEST_POLICY"
    echo "reason:    $RESOLVED_TEST_REASON"
  } > "$logdir/test-policy.log"
  [[ ${#warnings[@]} -gt 0 ]] && log "WARN: ${#warnings[@]} TEST_ELIGIBILITY warning(s) — see $logdir/test-policy.log"
  log "Resolved test policy: $RESOLVED_TEST_POLICY ($RESOLVED_TEST_REASON)"
}
read_targeted_test_target() {  # <file>
  [[ -f "$1" ]] || return 1
  local target
  target="$(head -n 1 "$1" | tr -d '\r')"
  [[ -n "$target" ]] || return 1
  printf '%s' "$target"
}
# Deterministic, TARGETED red->green gate for a SINGLE test. This is a targeted TDD gate,
# NOT a full no-regression gate: it proves only that the one designated test went red->green.
# Broad regression protection is whatever CHECKS[] already provides (see KNOWN LIMITATION in
# the generation rules below). Mode:
#   expect_red   (test-first phase) -> pass(0) iff the target test exits NON-ZERO, except 126/127
#                (not executable / command not found: the command is not runnable, never RED). This rejects a
#                tautological always-green test, but it does NOT prove the failure is an assertion
#                failure rather than a syntax/import/collection error: distinguishing those needs
#                framework-specific exit codes, which this stack-agnostic gate must not assume.
#                The "fails for the RIGHT reason" requirement is enforced by the test-authoring
#                PROMPT plus the mandatory expect_green pass on the SAME target (a test that was
#                red only from an import error must still be made to genuinely pass), NOT by exit
#                parsing. See KNOWN LIMITATION in the generation rules below.
#   expect_green (post-impl, default) -> ALWAYS reruns the target when one exists. If the hard
#                gate was armed (TEST_GATE_ACTIVE, set ONLY under `required`), pass(0) iff the
#                target test now PASSES. Under `preferred`, a still-red target is ADVISORY: it is
#                logged, fed into one check-fix attempt, and may still ship unresolved without
#                blocking the issue.
run_targeted_test_gate() {  # <logfile> [expect_red|expect_green]
  local logfile="$1" mode="${2:-expect_green}" target qtarget cmd rc=0 gate_mode="advisory"
  [[ "$mode" == "expect_green" && "$TEST_GATE_ACTIVE" == true ]] && gate_mode="hard"
  if [[ -z "$TARGETED_TEST_CMD" ]]; then
    echo "SKIP: no TARGETED_TEST_CMD configured" > "$logfile"; return 0
  fi
  if [[ "$mode" == "expect_green" && -n "$FROZEN_TARGETED_TEST_TARGET" ]]; then
    target="$FROZEN_TARGETED_TEST_TARGET"
  elif ! target="$(read_targeted_test_target "$TARGETED_TEST_FILE")"; then
    if [[ "$mode" == "expect_green" && "$gate_mode" != "hard" ]]; then
      echo "SKIP: no targeted test target written to $TARGETED_TEST_FILE (advisory mode)" > "$logfile"; return 0
    fi
    echo "FAIL: no targeted test target written to $TARGETED_TEST_FILE" > "$logfile"; return 1
  fi
  # SECURITY: the target is model-authored (lower trust than governance-authored CHECKS[])
  # and is substituted into a shell command run via `bash -c`. Allow only test-id/path
  # characters — reject anything that could inject shell. (']' first and '-' last keep the
  # bracket expr literal.) A leading '-' would be read as an option, a leading '#' as a
  # comment that drops the rest of the command. Sanitizing here is required regardless of
  # bash -c vs eval; the survivor is then substituted shell-QUOTED (printf %q), so '[', ']'
  # and '#' stay literal (no glob, no comment) — {TARGET} must stand unquoted in the command.
  if [[ ! "$target" =~ ^[][A-Za-z0-9_./:@=+#-]+$ || "$target" == [-#]* ]]; then
    if [[ "$mode" == "expect_green" && "$gate_mode" != "hard" ]]; then
      echo "ADVISORY: targeted test target '$target' contains disallowed characters or starts with '-'/'#'; preferred policy does not block" > "$logfile"; return 10
    fi
    echo "FAIL: targeted test target '$target' contains disallowed characters or starts with '-'/'#'" > "$logfile"; return 1
  fi
  qtarget="$(printf '%q' "$target")"
  cmd="${TARGETED_TEST_CMD//\{TARGET\}/"$qtarget"}"   # quoted replacement: literal under bash 5.2 patsub_replacement
  { echo "=== ($mode) $cmd ==="; bash -c "$cmd" 2>&1; } > "$logfile" || rc=$?   # bash -c (not eval); $target is allowlist-sanitized and quoted above
  if [[ "$mode" == "expect_red" && ( "$rc" -eq 126 || "$rc" -eq 127 ) ]]; then
    # Not executable / command not found is a broken command, never a RED proof.
    echo "NOT RED: targeted test command not runnable (exit $rc)" >> "$logfile"
    log "WARN: targeted test command not runnable (exit $rc); not accepted as RED."; return 1
  fi
  if [[ "$mode" == "expect_red" ]]; then
    [[ "$rc" -ne 0 ]] && {
      FROZEN_TARGETED_TEST_TARGET="$target"
      echo "RED OK (exit $rc): non-zero before implementation (reason not exit-verified — see expect_red note)" >> "$logfile"
      echo "LOCKED TARGET: $FROZEN_TARGETED_TEST_TARGET" >> "$logfile"
      return 0
    }
    echo "NOT RED (exit 0): target passes without implementation — tautological or behavior already exists" >> "$logfile"; return 1
  fi
  [[ "$rc" -eq 0 ]] && { echo "GREEN OK" >> "$logfile"; return 0; }
  if [[ "$gate_mode" == "hard" ]]; then
    echo "NOT GREEN (exit $rc)" >> "$logfile"; return 1
  fi
  echo "ADVISORY: target still red after implementation (exit $rc); preferred policy does not block" >> "$logfile"; return 10; }
ensure_checks_pass() {  # <issue> <logdir> <prefix>
  local tl="$2/$3-targeted-test.log" cl="$2/$3.log" combo="$2/$3-combined.log" tg=0 rc=0
  # Run BOTH (no &&-short-circuit): under `set -e` a skipped run_checks would leave $cl
  # missing and the later `cat "$cl"` would abort the script before auto-fix. `|| x=$?`
  # also keeps a failing check from tripping `set -e` while we capture its status.
  run_targeted_test_gate "$tl" expect_green || tg=$?
  run_checks "$cl" || rc=$?
  [[ "$tg" -eq 0 && "$rc" -eq 0 ]] && { log "Checks passed."; return 0; }
  if [[ "$tg" -eq 10 && "$rc" -eq 0 ]]; then
    log "Targeted test remains advisory under preferred policy; checks are already green, so no auto-fix runs."
    return 0
  else
    log "Checks failed; auto-fixing..."
  fi
  { cat "$tl" 2>/dev/null; echo; cat "$cl" 2>/dev/null; } > "$combo"
  build_check_fix_prompt "$1" "$(cat "$combo")" "$2/$3-fix-prompt.txt"
  # A crashed check-fix runner is only a WARN: the check re-run below decides.
  run_impl_model "$2/$3-fix-prompt.txt" > "$2/$3-fix.log" 2>&1 \
    || log "WARN: check-fix runner failed (exit $?); the check re-run decides."
  tg=0; rc=0
  run_targeted_test_gate "$2/$3-rerun-targeted-test.log" expect_green || tg=$?
  run_checks "$2/$3-rerun.log" || rc=$?
  [[ "$tg" -eq 0 && "$rc" -eq 0 ]] && { log "Checks pass after fix."; return 0; }
  if [[ "$tg" -eq 10 && "$rc" -eq 0 ]]; then
    log "Checks pass, but the targeted test remains red under preferred policy (advisory only)."
    return 0
  fi
  log "Checks still failing."; return 1; }

# --- Prompt builders: insert the bash blueprint from prompt-builders.md HERE, verbatim, filling
#     only its three placeholders (PROJECT_NAME, REFERENCE_DOCS, GOVERNANCE_REVIEW_FOCUS).
#     It defines build_implementation_prompt / build_test_authoring_prompt / build_review_prompt
#     / build_fix_prompt / build_refactor_prompt / build_memory_update_prompt
#     / build_check_fix_prompt with exactly the argument lists the call sites in this skeleton
#     use, and reads the per-task globals RESOLVED_SKILL*, RESOLVED_TEST_*, TARGETED_TEST_FILE,
#     FROZEN_TARGETED_TEST_TARGET, REVIEW_B_ENABLED and (local task list) TASK_FILE. Untrusted
#     values (title, body, diff, findings, check output) are written as data, never expanded. ---

# --- Deterministic skill resolution (see `automate.md` "Deterministic skill resolution" and `contract.md` section 4).
#     Resolve ONCE per task from SKILL_MAP, log searched/candidates/chosen/reason,
#     and let only the implement/fix/refactor prompts inject the result. Outcomes:
#       (none)       no matcher matched (or SKILL_MAP empty) -> nothing injected
#       <skill>      exactly one DISTINCT skill matched      -> injected
#       (ambiguous)  >1 distinct skills matched              -> nothing injected, logged
#     No registry, no network, no semantic fallback: ambiguity is preferred over guessing.
#     Resolution runs before implementation, so it never depends on post-impl changes. ---
# <labels> is a NEWLINE-separated list (a single label may contain spaces, e.g.
# "good first issue" or "area: auth"). Matchers are trimmed, so spaced policy entries work.
# Only label: and title: matchers exist — both resolve deterministically without touching
# the filesystem. (A path: matcher was intentionally dropped: it was a no-op in GitHub-issue
# mode and filesystem-dependent in task-list mode. Reintroduce only with an explicit design.)
resolve_skill() {  # <labels> <title> <body> <logdir>
  local labels="$1" title="$2" body="$3" logdir="$4"
  local text="$title"$'\n'"$body"
  local -a reasons=() distinct=() warnings=()
  local entry lhs type pat skill matched regex_rc l d seen
  for entry in "${SKILL_MAP[@]}"; do
    # Malformed entries are warned AND logged, never silently skipped or resolved to an empty skill.
    [[ "$entry" == *=* ]] || { warnings+=("malformed SKILL_MAP entry '$entry' (no '=<skill>') — ignored"); continue; }
    skill="$(trim "${entry##*=}")"; lhs="${entry%=*}"
    type="$(trim "${lhs%%:*}")"; pat="$(trim "${lhs#*:}")"
    [[ -n "$skill" ]] || { warnings+=("SKILL_MAP entry '$entry' has an empty skill name — ignored"); continue; }
    matched=false
    case "$type" in
      # Match each label as a WHOLE line; never word-split (would break labels with
      # spaces and could false-match a fragment of a multi-word label). Dead on a
      # label-less source or with an empty pattern: warned, never matched.
      label) if [[ "$TASK_SOURCE_HAS_LABELS" != true ]]; then
               warnings+=("dead label: matcher '$entry': this task source has no labels [GOVERNANCE DRIFT]")
             elif [[ -z "$pat" ]]; then
               warnings+=("dead label: matcher '$entry': empty pattern")
             else
               while IFS= read -r l; do [[ -n "$l" && "$l" == "$pat" ]] && matched=true; done <<< "$labels"
             fi ;;
      # An invalid ERE makes [[ =~ ]] return 2 (not 1). Don't let a typo silently
      # skip a configured skill with only noisy stderr: suppress the diagnostic,
      # detect rc 2, and record an explicit warning into skill-resolution.log.
      title) if [[ "$text" =~ $pat ]] 2>/dev/null; then
               matched=true
             else
               # shellcheck disable=SC2319  # intended: $? of the [[ =~ ]] test (2 = invalid regex)
               regex_rc=$?
               if [[ "$regex_rc" -eq 2 ]]; then
                 warnings+=("invalid title: regex '$pat' (entry '$entry') — treated as no match")
               fi
             fi ;;
      *)     warnings+=("unknown SKILL_MAP matcher type '$type' in '$entry' — ignored") ;;
    esac
    [[ "$matched" == true ]] || continue
    reasons+=("$type:$pat -> $skill")
    seen=false; for d in "${distinct[@]}"; do [[ "$d" == "$skill" ]] && seen=true; done
    [[ "$seen" == false ]] && distinct+=("$skill")
  done
  if [[ ${#distinct[@]} -eq 0 ]]; then
    RESOLVED_SKILL="(none)"; RESOLVED_SKILL_REASON="no matcher matched"
  elif [[ ${#distinct[@]} -eq 1 ]]; then
    RESOLVED_SKILL="${distinct[0]}"; RESOLVED_SKILL_REASON="$(IFS='; '; echo "${reasons[*]}")"
  else
    RESOLVED_SKILL="(ambiguous)"; RESOLVED_SKILL_REASON="multiple distinct skills: $(IFS=', '; echo "${distinct[*]}")"
  fi
  {
    echo "searched: labels=[$(printf '%s' "$labels" | tr '\n' ',')] title=[$title]"
    echo "candidates:"; printf '  - %s\n' "${reasons[@]:-(none)}"
    [[ ${#warnings[@]} -gt 0 ]] && { echo "warnings:"; printf '  - %s\n' "${warnings[@]}"; }
    echo "chosen:    $RESOLVED_SKILL"
    echo "reason:    $RESOLVED_SKILL_REASON"
  } > "$logdir/skill-resolution.log"
  [[ ${#warnings[@]} -gt 0 ]] && log "WARN: ${#warnings[@]} invalid or dead SKILL_MAP matcher(s) — see $logdir/skill-resolution.log"
  log "Resolved skill: $RESOLVED_SKILL ($RESOLVED_SKILL_REASON)"; }

# --- Code diff vs {{BASE_BRANCH}}, INCLUDING new files. CRITICAL: plain
#     `git diff <base>` omits UNTRACKED files, so a brand-new file from the
#     implementation would be invisible and reviewers would approve an empty diff.
#     Stage first (so new files register), then diff the index against base.
#     Excludes MEMORY.md via the `:!` pathspec. Logs never reach the index: stage_repo_changes
#     runs `git add -A` and then unstages $LOGDIR (which is usually gitignored as well). ---
stage_for_diff() { stage_repo_changes; }   # same staging; new files become visible to git diff
code_diff() { stage_for_diff; git diff --cached "$BASE_BRANCH" -- . ":!$MEMORY_FILE"; }
# Hash the code diff with `git hash-object` (git is already a hard dep), NOT `md5sum`:
# md5sum is absent by default on macOS/Windows, so under `set -euo pipefail` a generated
# script would die — less portable than the "bash + git + gh" contract claims. An empty
# diff hashes to the stable empty-blob id, so no-op detection still works.
code_hash() { stage_for_diff; git diff --cached "$BASE_BRANCH" -- . ":!$MEMORY_FILE" | git hash-object --stdin; }
# Whole-tree hash INCLUDING MEMORY.md (logs excluded as above): the reviewer tamper check. code_hash
# stays the M1/M4 measure (MEMORY.md excluded); a reviewer must not touch MEMORY.md either (M7).
tree_hash() { stage_repo_changes; git write-tree; }

# --- Review: uses code_diff (full uncommitted work vs {{BASE_BRANCH}} — new files
#     included, MEMORY.md excluded) so status churn is hidden but real changes are not ---
run_review() {  # <label> <runner_fn> <issue> <title> <body> <logfile> <logdir> -> 0 pass, 1 findings, 2 runner failed, 3 tree modified
  local diff stripped verdict re before after rrc=0; diff="$(code_diff 2>/dev/null || true)"
  [[ -z "$diff" ]] && { echo "LGTM (no changes)" > "$6"; return 0; }
  build_review_prompt "$1" "$3" "$4" "$5" "$diff" "$7/prompt-review.txt"
  before="$(tree_hash)" || before=""
  # stdout = the model's final message ONLY (runner contract above); stderr goes to a sidecar so
  # CLI progress/preamble never reaches the verdict scan. A crashed runner is rc 2, not a rejection.
  "$2" "$7/prompt-review.txt" > "$6" 2> "$6.stderr" || rrc=$?
  # Reviewers are read-only (their runners carry read-only flags; this is the backstop). Any change
  # to the tree, MEMORY.md included, cannot be undone cheaply: a failed review (rc 3, no retry), and
  # the issue fails via the normal path. An unhashable tree fails closed the same way.
  after="$(tree_hash)" || after=""
  [[ -n "$before" && "$after" == "$before" ]] || {
    echo "REVIEWER FAILED: reviewer modified the working tree; not a pass." >> "$6"
    log "ERROR: $1 modified the working tree during review."; REVIEW_TAMPERED=true; return 3; }
  [[ "$rrc" -eq 0 ]] || { echo "REVIEWER FAILED: runner exited $rrc; not a pass." >> "$6"; return 2; }
  # Verdict, conservative (the review prompt states the same format: EITHER LGTM, optionally followed
  # by ADVISORY: lines, OR numbered findings — never both). After stripping markdown decoration per
  # line, (1) the FIRST decisive line (LGTM or a numbered finding "1." / "1)") must be LGTM alone or
  # LGTM + a separator (. ! : ; , ( -), AND (2) no numbered finding line may appear anywhere.
  # "LGTM", "**LGTM**", "LGTM (no changes)", LGTM + ADVISORY: lines pass; "LGTM must not be granted",
  # "LGTM? Not yet.", findings-first replies and LGTM + a numbered finding fail.
  # shellcheck disable=SC2016  # the backticks are literal markdown characters, not an expansion
  stripped="$(sed -E 's/^[[:space:]>*_`#-]+//; s/[[:space:]*_`]+$//' "$6")"
  verdict="$(grep -m1 -iE '^(LGTM|[0-9]+[.)])' <<< "$stripped" || true)"
  re='^[Ll][Gg][Tt][Mm]([[:space:]]*[.!:;,(-].*)?$'
  [[ "$verdict" =~ $re ]] || return 1
  ! grep -qE '^[0-9]+[.)]' <<< "$stripped"; }   # LGTM and no numbered finding = pass

# --- Review-until-pass: A/B review + fix loop with no-op detection. With REVIEW_B_ENABLED=false
#     Reviewer B is not run and counts as passed (single review), and the fix prompt gets no B log.
#     Reused for BOTH the correctness pass and the refactor re-validation.
#     Sets REVIEW_ROUNDS (final round) and REVIEW_OUTCOME (globals):
#       clean         = every enabled reviewer passed
#       accepted-noop = reviewers had findings but a fix cycle changed no code
#                       (tolerated by the correctness pass to avoid infinite loops)
#       failed        = rounds exhausted, a check failed, a reviewer/fix runner crashed, or a
#                       reviewer modified the working tree
#     Returns 0 for clean OR accepted-noop, 1 for failed. Callers that must NOT
#     tolerate unresolved findings (the refactor pass) check REVIEW_OUTCOME == clean,
#     not just the exit code. ---
# One reviewer call with a single retry on RUNNER failure (rc 2 = the CLI crashed, e.g. rate
# limit — not a rejection). Returns 0 pass, 1 findings, 2 failed twice, 3 tree modified (no retry).
run_review_retry() {  # same args as run_review
  local rc=0; run_review "$@" || rc=$?
  [[ "$rc" -eq 2 ]] || return "$rc"
  log "WARN: $1 runner failed; retrying once in 15s."; sleep 15
  rc=0; run_review "$@" || rc=$?
  [[ "$rc" -eq 2 ]] && log "ERROR: $1 runner failed twice; failing the issue."
  return "$rc"; }
review_until_pass() {  # <issue> <title> <body> <logdir> <stage>
  local issue="$1" title="$2" body="$3" logdir="$4" stage="$5" round=1
  REVIEW_ROUNDS=0; REVIEW_OUTCOME=failed; REVIEW_TAMPERED=false
  while [[ "$round" -le "$MAX_ROUNDS" ]]; do
    REVIEW_ROUNDS="$round"
    local a="$logdir/$stage-rev-a-r$round.log" b="" ap=true bp=true rc=0
    # A crashed reviewer (rc 2) or one that edited the tree (rc 3) FAILS the issue: it must never
    # turn into a fix round whose no-op is then accepted as "remaining findings" (unreviewed code).
    run_review_retry "$REVIEW_A_LABEL" run_review_a_model "$issue" "$title" "$body" "$a" "$logdir" || rc=$?
    [[ "$rc" -ge 2 ]] && { REVIEW_OUTCOME=failed; return 1; }; [[ "$rc" -eq 0 ]] || ap=false
    if [[ "$REVIEW_B_ENABLED" == true ]]; then   # single review: B never runs, bp stays true, b stays ""
      b="$logdir/$stage-rev-b-r$round.log"; rc=0
      run_review_retry "$REVIEW_B_LABEL" run_review_b_model "$issue" "$title" "$body" "$b" "$logdir" || rc=$?
      [[ "$rc" -ge 2 ]] && { REVIEW_OUTCOME=failed; return 1; }; [[ "$rc" -eq 0 ]] || bp=false
    fi
    [[ "$ap" == true && "$bp" == true ]] && { REVIEW_OUTCOME=clean; return 0; }
    [[ "$round" -ge "$MAX_ROUNDS" ]] && { REVIEW_OUTCOME=failed; return 1; }
    local before after
    before="$(code_hash)"
    build_fix_prompt "$issue" "$title" "$body" "$a" "$b" "$round" "$logdir/$stage-fix-r$round.txt"
    # A crashed fixer is not a deliberate no-op: fail instead of accepting the open findings.
    run_impl_model "$logdir/$stage-fix-r$round.txt" > "$logdir/$stage-fix-r$round.log" 2>&1     || { log "ERROR: fix runner failed (round $round)"; REVIEW_OUTCOME=failed; return 1; }
    after="$(code_hash)"
    [[ "$before" == "$after" ]] && {
      log "No code change in fix cycle; remaining findings accepted."; REVIEW_OUTCOME=accepted-noop; return 0; }
    ensure_checks_pass "$issue" "$logdir" "$stage-checks-r$round" || { REVIEW_OUTCOME=failed; return 1; }
    round=$((round + 1))
  done
  REVIEW_OUTCOME=failed; return 1; }

# --- Refactor stage: SECOND pass. Runs on top of the COMMITTED correctness state
#     (HEAD is the per-issue checkpoint commit). Asks the impl model to simplify to
#     senior-engineer quality WITHOUT changing behavior, then re-validate via review_until_pass.
#     A round is KEPT (amended into the checkpoint) only when its re-review is CLEAN.
#     If a round fails checks, fails review, OR its fix cycle no-ops while findings
#     remain (REVIEW_OUTCOME != clean), that round is REVERTED to the checkpoint and
#     refactoring stops — a degraded refactor is never accepted, and the already-
#     approved correctness work is never lost. Stops on no-op (converged) or
#     MAX_REFACTOR_ROUNDS. Accumulates DELIVERED_REVIEW_ROUNDS / REFACTOR_ROUNDS for
#     accurate commit + memory metadata. Same MEMORY.md/logs exclusions.
#     Returns 1 (the caller fails the issue, checkpoint kept) only for an M7 violation: the
#     round changed SOUL.md/AGENTS.md/CLAUDE.md, or a reviewer modified the working tree. ---
refactor_stage() {  # <issue> <title> <body> <logdir>
  [[ "$REFACTOR" == true ]] || { log "Refactor pass disabled (--no-refactor)."; return 0; }
  local issue="$1" title="$2" body="$3" logdir="$4" r=1 before after ok violated
  while [[ "$r" -le "$MAX_REFACTOR_ROUNDS" ]]; do
    REVIEW_TAMPERED=false; violated=false
    before="$(code_hash)"
    build_refactor_prompt "$issue" "$title" "$body" "$r" "$logdir/05-refactor-r$r.txt"
    # A crashed refactor runner is NOT "converged" (its hash may be unchanged): revert and stop.
    if ! run_impl_model "$logdir/05-refactor-r$r.txt" > "$logdir/05-refactor-r$r.log" 2>&1; then
      log "WARN: refactor runner failed in round $r; reverting this round to the checkpoint and stopping refactoring."
      git reset --hard HEAD >/dev/null 2>&1; git clean -fd -e "$LOGDIR" >/dev/null 2>&1
      return 0
    fi
    after="$(code_hash)"
    [[ "$before" == "$after" ]] && { log "Refactor round $r: no change; code already clean."; return 0; }
    log "Refactor round $r changed code; re-checking and re-reviewing."
    ok=true
    ensure_checks_pass "$issue" "$logdir" "05-refactor-checks-r$r" || ok=false
    if [[ "$ok" == true ]]; then
      review_until_pass "$issue" "$title" "$body" "$logdir" "06-refactor-rev-r$r" || ok=false
      [[ "$REVIEW_OUTCOME" == clean ]] || ok=false   # no-op-accept is NOT good enough for a refactor
    fi
    [[ "$REVIEW_TAMPERED" == true ]] && violated=true
    if [[ "$ok" == true ]] && ! governance_untouched; then
      log "ERROR: refactor round $r changed SOUL.md/AGENTS.md/CLAUDE.md (M7)."; ok=false; violated=true
    fi
    if [[ "$ok" != true ]]; then
      log "Refactor round $r not cleanly approved; reverting this round to the checkpoint and stopping."
      git reset --hard HEAD >/dev/null 2>&1   # back to the checkpoint (correctness + prior clean refactors)
      git clean -fd -e "$LOGDIR" >/dev/null 2>&1   # drop files the bad round added; keep this issue's logs
      [[ "$violated" == true ]] && return 1
      return 0
    fi
    # Accepted: fold this cleanly-reviewed refactor into the checkpoint commit. Guarded: a failed
    # fold would leave the round uncommitted for a later revert to drop while the counters still
    # reported it, so revert it now and stop; the counters are bumped only after the fold.
    stage_repo_changes
    git commit --amend --no-edit >/dev/null     || { log "WARN: could not fold refactor round $r into the checkpoint; reverting it and stopping."; git reset --hard HEAD >/dev/null 2>&1; git clean -fd -e "$LOGDIR" >/dev/null 2>&1; return 0; }
    DELIVERED_REVIEW_ROUNDS=$((DELIVERED_REVIEW_ROUNDS + REVIEW_ROUNDS))
    REFACTOR_ROUNDS="$r"
    r=$((r + 1))
  done
  log "Reached MAX_REFACTOR_ROUNDS ($MAX_REFACTOR_ROUNDS); accepting current state."
  return 0; }

# --- Per-issue pipeline ---
# Returns 0 done, 1 failed, 3 skipped (awaiting review), 4 dry run (would process).
process_issue() {  # <issue>
  local issue="$1" title body labels logdir branch out arc=0
  # All task-source reads are guarded and happen BEFORE any git mutation, so a plain `return 1` is
  # safe here (an unreadable issue must not run the pipeline with an empty title/body/labels).
  title="$(gh issue view "$issue" --json title --jq '.title')"     || { log "ERROR: cannot read #$issue (gh failed)"; return 1; }
  body="$(gh issue view "$issue" --json body --jq '.body')"        || { log "ERROR: cannot read #$issue (gh failed)"; return 1; }
  # Newline-join so multi-word labels ("good first issue") stay one token (see resolve_skill).
  labels="$(gh issue view "$issue" --json labels --jq '[.labels[].name]|join("\n")')"     || { log "ERROR: cannot read #$issue (gh failed)"; return 1; }
  branch="issue-$issue-$(slugify "$title")"
  # End of the task-source reads (the local task-list variant replaces the block above).
  branch_awaiting_review "$branch" || arc=$?
  case "$arc" in
    0) log "Skip #$issue: $AWAIT_REASON — awaiting review / finish manually (push and open the PR, or merge it), or delete branch $branch to redo it."; return 3 ;;
    2) log "ERROR: cannot check $branch for own commits / an open PR (git or gh failed)."; return 1 ;;
  esac
  [[ "$DRY_RUN" == true ]] && { log "[dry-run] would process #$issue: $title (branch $branch)"; return 4; }
  require_clean_worktree || return 1
  refresh_base || { log "ERROR: cannot refresh $BASE_BRANCH before #$issue."; return 1; }
  logdir="$(ensure_logdir "$issue")"
  # Check out OUTSIDE a `$(...)`: a substitution would swallow a failed checkout and the run
  # would continue — and commit — on the base branch.
  create_issue_branch "$branch" || { log "ERROR: cannot check out $branch"; return 1; }
  log "Branch: $branch"
  TARGETED_TEST_FILE="$logdir/targeted-test.txt"
  rm -f "$TARGETED_TEST_FILE"
  FROZEN_TARGETED_TEST_TARGET=""
  TEST_GATE_ACTIVE=false

  # 0. Resolve the designated skill ONCE for this task (deterministic; logged).
  #    Globals RESOLVED_SKILL / RESOLVED_SKILL_REASON are then injected by the
  #    implement/fix/refactor prompt builders. Issue mode matches on labels; the local
  #    task-list variant (no labels) relies on title: matchers against the task title.
  resolve_skill "$labels" "$title" "$body" "$logdir"
  resolve_test_policy "$labels" "$title" "$body" "$logdir"

  # 0a. Test-first (RED) sub-phase — only when a targeted test command exists and the task is
  #     test-eligible. Author ONLY the test(s), prove the target is RED before any implementation.
  #     The HARD red->green gate (TEST_GATE_ACTIVE -> enforced by ensure_checks_pass) is armed
  #     ONLY under `required`: a confirmed RED then makes the post-impl green a blocking gate.
#     Under `preferred` the test still ships and is rerun post-implementation, but green remains
#     ADVISORY — `preferred` must never hard-block or discard correctness work (that is what
#     `required` is for; see the asymmetric review channel). A confirmed RED target is FROZEN for
#     the rest of the task, so later prompts cannot retarget the gate to a different test. When
#     checks are already green, an advisory-only targeted-test miss is logged and returned as-is;
#     the pipeline must not mutate code just to chase an optional green. For `required`, an
#     unprovable RED fails the issue; `preferred` always continues with advisory guidance only.
  if [[ "$RESOLVED_TEST_POLICY" != "off" && -n "$TARGETED_TEST_CMD" ]]; then
    build_test_authoring_prompt "$issue" "$title" "$body" "$logdir/prompt-test.txt"
    run_impl_model "$logdir/prompt-test.txt" > "$logdir/00-test-author.log" 2>&1 \
      || { return_to_base; return 1; }
    if run_targeted_test_gate "$logdir/00-test-red.log" expect_red; then
      if [[ "$RESOLVED_TEST_POLICY" == "required" ]]; then
        TEST_GATE_ACTIVE=true
        log "RED confirmed before implementation; hard red->green gate is active (required)."
      else
        log "RED confirmed; preferred — the test ships and post-impl green is ADVISORY (no hard gate)."
      fi
    elif [[ "$RESOLVED_TEST_POLICY" == "required" ]]; then
      log "[TEST GATE] required: no RED baseline (missing/invalid target, test already green, or command not runnable); cannot prove red->green. Failing issue."
      return_to_base; return 1
    else
      log "[TEST GATE] preferred: no RED baseline; continuing with advisory test guidance only (no hard gate)."
    fi
  fi

  # 1. Implement
  build_implementation_prompt "$issue" "$title" "$body" "$logdir/prompt-impl.txt"
  run_impl_model "$logdir/prompt-impl.txt" > "$logdir/01-impl.log" 2>&1 \
    || { return_to_base; return 1; }
  # 2. Checks
  ensure_checks_pass "$issue" "$logdir" "02-checks" || { return_to_base; return 1; }

  # 3. Correctness review loop (A, plus B when enabled) with no-op fix detection
  review_until_pass "$issue" "$title" "$body" "$logdir" "03" || { return_to_base; return 1; }
  local review_rounds="$REVIEW_ROUNDS"
  DELIVERED_REVIEW_ROUNDS="$review_rounds"   # globals; refactor_stage accumulates accepted refactor re-reviews into these
  REFACTOR_ROUNDS=0

  # 3a. Checkpoint the correctness-approved state as the issue commit. This makes the
  #     refactor pass safe: a bad refactor reverts to THIS commit (correctness work is
  #     never lost), and only cleanly re-reviewed refactors are amended in. Gate on a
  #     non-empty CODE diff (excludes MEMORY.md/logs), NOT has_repo_changes_outside_logs:
  #     run_review auto-LGTMs an empty code diff, so a model that only rewrote the
  #     MEMORY.md "Next Up" line would otherwise pass review and produce a memory-only
  #     "implemented" commit + PR. No code change => nothing was built => skip the issue.
  [[ -n "$(code_diff)" ]] || { log "No code changes for #$issue (only MEMORY.md/logs)."; return_to_base; return 1; }
  governance_untouched || { log "ERROR: #$issue changed SOUL.md/AGENTS.md/CLAUDE.md (M7: read-only at runtime); discarding."; return_to_base; return 1; }
  stage_repo_changes
  # Explicit guard: process_issue runs inside `&& ... ||` at the call site, where bash suspends
  # `set -e` for the whole function body. Without it a failed checkpoint (hook, signing,
  # identity) would let refactor_stage `reset --hard HEAD` onto the BASE tip and discard the
  # approved correctness work. git's own reason is logged (it goes to stdout for "nothing to
  # commit"), and the approved diff is saved under $LOGDIR (survives return_to_base) first.
  out="$(git commit -m "feat: implement #$issue - $title (correctness)" 2>&1)"     || { log "ERROR: checkpoint commit failed for #$issue: ${out//$'\n'/ | }"; git diff --cached "$BASE_BRANCH" -- . ":!$MEMORY_FILE" > "$logdir/approved-uncommitted.patch"; log "Approved diff saved to $logdir/approved-uncommitted.patch; discarding and skipping."; return_to_base; return 1; }

  # 3b. Refactor stage: simplify to senior quality on top of the checkpoint, re-validated
  #     by the reviewers. Reverts any round that is not cleanly approved; fails the issue only on an M7
  #     violation (governance edit, or a reviewer that modified the tree).
  #     Skipped entirely with --no-refactor.
  refactor_stage "$issue" "$title" "$body" "$logdir" \
    || { log "ERROR: refactor pass of #$issue violated M7 (round reverted); checkpoint kept on $branch."; return_to_base; return 1; }

  # 4. Memory step OWNS completed-work: archive entry + clear Next Up line.
  #     Pass the correctness FIX rounds and the accepted REFACTOR rounds SEPARATELY —
  #     they mean different things. "Last fix" is gated on fix rounds (review_rounds),
  #     the simplification note on refactor rounds. Do not conflate them into one number.
  build_memory_update_prompt "$issue" "$title" "$review_rounds" "$REFACTOR_ROUNDS" "$logdir/prompt-mem.txt"
  # Guarded: a crashed memory step would otherwise ship a PR with a stale $MEMORY_FILE and no
  # archive entry (contract M3) without a trace.
  run_impl_model "$logdir/prompt-mem.txt" > "$logdir/07-mem.log" 2>&1     || { log "ERROR: memory step failed for #$issue; checkpoint kept on $branch."; return_to_base; return 1; }
  governance_untouched || { log "ERROR: governance files changed after the checkpoint of #$issue (M7); uncommitted changes dropped, checkpoint kept on $branch."; return_to_base; return 1; }

  # 5. Fold memory + the final message into the ONE issue commit, then PR.
  #     (merge ONLY if user opted in — otherwise stop here for human review)
  stage_repo_changes
  local refactor_summary="off"
  [[ "$REFACTOR" == true ]] && refactor_summary="on (${REFACTOR_ROUNDS} round(s) applied)"
  git commit --amend -m "feat: implement #$issue - $title

Automated via auto-develop.sh. Model plan: $IMPL_LABEL | $REVIEW_PLAN
Correctness review rounds: $review_rounds/$MAX_ROUNDS (delivered review rounds incl. accepted refactor re-reviews: $DELIVERED_REVIEW_ROUNDS)
Refactor pass: $refactor_summary

Closes #$issue" >/dev/null     || { log "ERROR: final commit --amend failed for #$issue; checkpoint kept on $branch."; return_to_base; return 1; }
  # Same `set -e` caveat: a failed push or PR must not fall through to "Done" and be counted
  # as completed. Everything is committed on the issue branch, which is kept for manual handling.
  git push -u origin "$branch"     || { log "ERROR: push failed for #$issue; branch $branch kept locally."; return_to_base; return 1; }
  gh pr create --title "#$issue: $title" --body "Closes #$issue. Logs: \`$logdir/\`"     || { log "ERROR: PR creation failed for #$issue; branch $branch is pushed, open the PR manually."; return_to_base; return 1; }
  # Return to BASE_BRANCH so the NEXT issue branches from a clean base rather than stacking
  # on this still-unmerged branch. Safe: everything is committed at this point.
  git checkout "$BASE_BRANCH" >/dev/null 2>&1 || log "WARN: could not return to $BASE_BRANCH"
  # Auto-merge ONLY when the operator opted in (and confirmed); otherwise stop at the
  # PR for human review. Already on $BASE_BRANCH, so a successful merge + pull is safe.
  if [[ "$AUTO_MERGE" == true ]]; then
    # A failed merge/pull must NOT count as a completed issue (matches the sample fixture):
    # report the failure so the loop's `|| log "...failed"` fires and the issue is not counted.
    if ! { gh pr merge "$branch" --squash && git pull; }; then
      log "ERROR: auto-merge/pull failed for #$issue; PR left open for manual handling."
      return 1
    fi
  else
    log "PR opened for #$issue; auto-merge OFF (enable with --auto-merge). Awaiting human review."
  fi
  log "=== Done #$issue ==="; }

# --- Candidate selection + main loop ---
# Start from a fresh state: clean tree, on BASE_BRANCH, fast-forwarded to its upstream.
# A dry run stays read-only and only reports what a real run would refuse.
if [[ "$DRY_RUN" == true ]]; then
  has_repo_changes_outside_logs && log "[dry-run] NOTE: working tree dirty outside $LOGDIR; a real run would refuse to start."
else
  require_clean_worktree || die "Commit or stash changes before running."
  refresh_base || die "Cannot start from a fresh $BASE_BRANCH."
fi

CANDIDATES=()
if [[ -n "$TARGET_ISSUE" ]]; then
  state="$(gh issue view "$TARGET_ISSUE" --json state --jq '.state')" || die "Cannot read #$TARGET_ISSUE (gh failed)."
  [[ "$state" == OPEN ]] || die "#$TARGET_ISSUE is $state, not OPEN."
  issue_has_label "$TARGET_ISSUE" || die "#$TARGET_ISSUE lacks {{TASK_LABEL}} (or gh failed)."
  CANDIDATES=("$TARGET_ISSUE")
else
  # Capture first: a failing `gh` inside `< <(...)` would be swallowed as "No eligible issues."
  list="$(gh issue list --label "{{TASK_LABEL}}" --state open --limit 200 --json number --jq '.[].number')" \
    || die "Cannot list issues (gh failed); check gh auth and network."
  mapfile -t CANDIDATES < <(printf '%s\n' "$list" | awk 'NF' | sort -n)
fi
[[ ${#CANDIDATES[@]} -eq 0 ]] && { log "No eligible issues."; exit 0; }

COMPLETED=0; SKIPPED=0; WOULD=0
for issue in "${CANDIDATES[@]}"; do
  [[ $((COMPLETED + WOULD)) -ge "$MAX_ISSUES" ]] && break
  check_dependencies "$issue" || { log "Skip #$issue (deps open or unverifiable)."; continue; }
  # NOTE: inside this `|| rc=$?` bash suspends `set -e` for the whole process_issue body (and
  # everything it calls). The guarded steps are: the task-source reads, the awaiting-review check,
  # the base refresh, the branch checkout, the impl/test/fix/memory runners, the reviewer runners
  # (rc 2 + one retry, rc 3 tree modified), the M7 checks, the checkpoint commit, the refactor
  # fold, the final amend, the push and the PR — each with its own `|| { log ...; return 1; }`
  # (plus return_to_base where work must be dropped).
  rc=0; process_issue "$issue" || rc=$?
  case "$rc" in
    0) COMPLETED=$((COMPLETED + 1)) ;;
    3) SKIPPED=$((SKIPPED + 1)) ;;
    4) WOULD=$((WOULD + 1)) ;;
    *) log "#$issue produced no changes/failed." ;;
  esac
done
if [[ "$DRY_RUN" == true ]]; then
  log "Dry run: would process $WOULD issue(s); skipped $SKIPPED awaiting review. Nothing was changed."
else
  log "Done. Completed $COMPLETED issue(s); skipped $SKIPPED awaiting review."
fi
```

## Generation rules

- **Keep the guards.** `require_clean_worktree`, dependency blocking, and the `return_to_base` rollback on failure are what make the loop safe to re-run. The rollback must **discard** in-progress work (`git reset --hard` + `git clean -fd -e "$LOGDIR"`, which keeps this issue's freshly written failure logs) before switching back, never a bare `git checkout "$orig"` — a half-written impl/fix would otherwise block the checkout or follow onto the base branch and trip the next run's clean-worktree guard.
- **Keep the explicit failure guards.** Bash suspends `set -e` inside `process_issue` and `check_dependencies` because the main loop calls them from `&& ... ||` lists, and never inherits it into `$(...)`. Every critical step therefore carries its own guard: the `gh` issue reads (title/body/labels, before any git mutation), the branch checkout (`create_issue_branch "$branch" || ...`, never inside a `$(...)`), the impl/test/fix/memory runners, the reviewer runners (`run_review` returns 2 on a crashed runner; `run_review_retry` retries once after 15 s, then the issue fails — a crash must never become a fix round whose no-op is accepted), a reviewer that modified the working tree (`run_review` compares `tree_hash` — the whole staged tree, `MEMORY.md` included, logs excluded — before and after the runner and returns 3, no retry; the issue fails, in the refactor pass after reverting the round), the M7 checks (`governance_untouched` before the checkpoint, before a refactor round is folded, and before the final amend; a violation fails the issue, after the checkpoint with the branch kept, and a refactor round is reverted first), the checkpoint commit (logs git's reason, saves the approved diff to `$logdir/approved-uncommitted.patch`, then discards), the refactor runner (a crash reverts the round to the checkpoint and stops refactoring; it is never read as "converged"), the refactor fold (`git commit --amend` reverts the round on failure, before the counters are bumped), the final amend, `git push`, and `gh pr create` (these keep the committed issue branch and drop only leftovers). A crashed check-fix runner only logs a WARN, because the check re-run decides. `check_dependencies` fails closed when `gh` cannot read the issue or a dependency, parses every `#N` on a `Depends on` line, and blocks (with the log "cross-repository dependency not supported; treating as blocked") on a cross-repository reference such as `owner/repo#13` or an issue/PR URL, which is never read as the local `#13`. The candidate listing captures the `gh issue list` output first and dies on a `gh` failure (a failing command inside `< <(...)` would read as "No eligible issues."). Do not generate a script that relies on `set -e` for these steps.
- **Start fresh, branch from the base branch, and return to it after every issue.** The script `cd`s to the repository root at startup (`code_diff`'s `-- .` pathspec and the clean-worktree guard are root-relative) and re-execs itself in tmux as `bash <absolute path>`. Before the first issue and before each issue branches, `refresh_base` checks out `{{BASE_BRANCH}}` and fast-forwards it to its upstream (`git fetch` + `git merge --ff-only`; no upstream is a no-op; a failure fails the issue, never forced), so a dependency merged on GitHub is present locally. `create_issue_branch <branch>` then starts the issue from `{{BASE_BRANCH}}` with `git checkout -B "$b" "$BASE_BRANCH"`, which also recreates a leftover branch without own commits from the current base tip. A branch **with** own commits ahead of the base, or (GitHub variant) an open PR for the branch, is finished work awaiting review: `branch_awaiting_review` makes `process_issue` return 3 and the issue is **skipped** with a log naming the branch (push/open the PR, or delete the branch) — not failed, not counted. Reprocessing it would stack a second run on top, and under `TEST_POLICY=required` the test-first phase would fail forever with NOT RED. `create_issue_branch` refuses to reset a branch with own commits. It only checks out and returns the checkout status; the caller computes the name and guards the call. The success path must `git checkout "$BASE_BRANCH"` after opening the PR. Otherwise, in a `--max-issues > 1` run, issue N branches off issue N-1's still-unmerged tip and its review diff includes the previous issue's code.
- **Gate the correctness commit on a non-empty CODE diff**, not on `has_repo_changes_outside_logs`. `run_review` auto-approves an empty `code_diff`, so a model that only rewrote the `{{MEMORY_FILE}}` "Next Up" line would otherwise pass review and produce a memory-only "implemented" commit + PR. Use `[[ -n "$(code_diff)" ]]` (excludes `{{MEMORY_FILE}}`/logs) as the checkpoint guard — no code change means nothing was built.
- **Review/no-op diffs must include new files.** Build every review diff and no-op hash from a *staged* diff (`stage_for_diff` + `git diff --cached <base>`), never plain `git diff <base>` — the latter omits untracked files, so a brand-new implementation file would be reviewed as an empty diff and silently approved. Use the `code_diff` / `code_hash` helpers everywhere a code diff is needed.
- **Stage with `git add -A` + unstage `$LOGDIR`, not `git add -- . :(exclude)$LOGDIR`.** When the log dir is gitignored (the recommended setup), the `:(exclude)` pathspec makes `git add` exit 1 on the matched-but-ignored path, which kills the run under `set -e`. Plain `git add -A` skips ignored paths silently; follow with `git reset -q -- "$LOGDIR"` to keep logs out of commits when they are *not* ignored.
- **`--dry-run` is side-effect-free.** It may select and print candidate work but must never mutate tracked files — no base refresh, no branch, no task-status flip, no commit, no PR. `process_issue` returns 4 right after the reads and the awaiting-review check; the loop counts these in `WOULD` and ends with "Dry run: would process N issue(s)", never "Completed". In the local task-list variant, `task_mark_status` (and any task-file write) is guarded by `[[ "$DRY_RUN" != true ]]`; a dirtied task file would trip the next run's clean-worktree guard.
- **Memory rules are mandatory** (see `contract.md` section 3, *Memory rules*): the review diff must exclude `{{MEMORY_FILE}}`; the no-op `code_hash` comparison must exclude `{{MEMORY_FILE}}` and `$LOGDIR`; only `build_memory_update_prompt` writes the archive entry. The reviewer tamper check is the one place that must **include** `{{MEMORY_FILE}}`: it uses `tree_hash` (`stage_repo_changes` + `git write-tree`), never `code_hash`, because a reviewer may not write `MEMORY.md` either.
- **Diff against `{{BASE_BRANCH}}`, not `BASE..HEAD`** — uncommitted fix-cycle changes must be visible to reviewers or the loop never converges.
- **Share the review loop.** `review_until_pass` is the single A/B-review-plus-fix implementation; the correctness pass and the refactor re-validation both call it. Do not duplicate the review/no-op logic. It exposes `REVIEW_OUTCOME` (`clean` / `accepted-noop` / `failed`) so callers can tell a clean pass from a tolerated no-op.
- **Single review is a toggle, not deleted code.** For a single-review project set `REVIEW_B_ENABLED=false` (placeholder legend); never strip the Reviewer B lines. The startup check accepts only `true`/`false`. With `false`: Reviewer B is never run and counts as passed in `review_until_pass`; the fix prompt gets an empty B log path and the prompt builders emit no Reviewer B text; the commit message's model plan (`REVIEW_PLAN`) omits it; the PATH preflight skips `REVIEW_B_RUNNER`; the tmux re-exec passes no `--review-b*`; and `--review-b` / `--review-b-effort` stay parseable but are **rejected** at startup with a clear message (a silently ignored reviewer flag would suggest a second review that never happens). The `REVIEW_B_*` variables stay defined (empty is fine), so nothing is unbound under `set -u`.
- **Checkpoint the correctness state before refactoring.** The correctness-approved work is committed (the per-issue checkpoint) *before* the refactor pass runs. This is what makes the second pass safe and is mandatory: a refactor that is not cleanly approved reverts to the checkpoint (`git reset --hard HEAD` + `git clean -fd -e "$LOGDIR"`), so correctness work is never lost and a degraded refactor is never kept. Cleanly-reviewed refactors are folded in with `git commit --amend`; the final commit message is set in one closing amend.
- **Refactor stage is gated, bounded, and never silently degrades.** It runs only after correctness passes, only when `REFACTOR=true`, and stops when a round changes nothing (`code_hash` no-op = converged) or `MAX_REFACTOR_ROUNDS` is reached. A round is **kept only when its re-review is `clean`** — failing checks, failing review, or a no-op fix cycle with findings still open (`REVIEW_OUTCOME != clean`) reverts that round; a crashed refactor runner reverts the round and stops (never read as "converged"). An M7 violation in a round (a governance file changed, or a reviewer modified the tree) reverts the round and fails the issue with the checkpoint branch kept. It is behavior-preserving simplification only — never a place to add features. `--no-refactor` must cleanly skip it.
- **Report the real history, with the right semantics.** The commit message reports the delivered review rounds (`DELIVERED_REVIEW_ROUNDS`: correctness plus accepted refactor re-reviews, not discarded refactor attempts) and `REFACTOR_ROUNDS`. The memory archive gets the correctness **fix** rounds (`review_rounds`) and `REFACTOR_ROUNDS` as *separate* arguments — never a conflated total — because "last fix" and "refactor rounds" are different facts; passing delivered review rounds into the "last fix" slot would imply fixes that never happened. `MEMORY.md` is part of the governance contract, so this accuracy is mandatory.
- **Pipe prompts via stdin** in the generated runner functions to avoid "Argument list too long" on large diffs.
- **Reviewer runners emit only the final message, read-only.** `run_review` scans the runner's stdout for the verdict and writes stderr to `<logfile>.stderr`; a reviewer runner must therefore print nothing but the model's final message on stdout and must not `2>&1`. `claude -p --output-format text` already does; `codex exec` prints a header, the echoed prompt and reasoning summaries, so run it with `--output-last-message <file>` and `cat` that file (or `--json` and extract the last agent message with jq). Any numbered line in such noise would otherwise read as a finding and fail every review. Reviewer runners carry **literal** read-only flags and never reference `$CLAUDE_PERMISSION_MODE` / `$CODEX_SANDBOX_MODE`: `claude` with `--permission-mode default --disallowedTools "Edit Write NotebookEdit Bash"`, `codex` with `--sandbox read-only`. The pass rule (the review prompt in `prompt-builders.md` states the same format: `LGTM` optionally followed by `ADVISORY:` lines, or numbered findings, never both): after stripping markdown decoration per line, the first decisive line (the first line that is `LGTM` or a numbered finding) must be `LGTM` alone or `LGTM` followed by a separator (`. ! : ; , ( -`), **and** no line anywhere may be a numbered finding (`^[0-9]+[.)]`); `ADVISORY:` lines are allowed. A runner exit status other than 0 is never a pass.
- **Headless permissions** (documentation only; the defaults stay safe). `claude -p` cannot answer permission prompts, so under the safe default `--permission-mode default` every tool that is not pre-allowed (Edit, Write, Bash for the checks) is denied, the implement/fix steps change nothing, and every task ends at the M6 "no code changes" gate. The operator pre-allows the needed tools in the target project's `.claude/settings.json` (`permissions.allow`) or passes `--allowedTools` on the implementer runner; `--unattended` stays the explicit, confirmed opt-in for `bypassPermissions` (claude implementer) or `danger-full-access` (codex implementer) and widens **only the implementer**. Reviewers are read-only independent of `--unattended`: a `claude` reviewer always runs with `--permission-mode default --disallowedTools "Edit Write NotebookEdit Bash"` (the diff is in the prompt; read-only tools such as Read/Grep/Glob stay available), a `codex` reviewer always with `--sandbox read-only`. The `tree_hash` check in `run_review` is the backstop, not the primary control.
- **Stdlib only** — bash + `git` + `gh` plus only the model CLIs the user explicitly selected. No extra deps unless governance lists them. In particular, hash code diffs with `git hash-object --stdin` (git is already required), **not** `md5sum`/`md5` — those are absent by default on macOS/Windows and would make the script die under `set -euo pipefail`.
- **Privileged flags off by default, behind a runtime confirmation.** Generated scripts must ship safe defaults (`CLAUDE_PERMISSION_MODE="default"`, `CODEX_SANDBOX_MODE="workspace-write"`, `AUTO_MERGE=false`) and reach privileged modes only via the `--unattended` / `--auto-merge` flags. Do **not** hardcode `bypassPermissions` / `danger-full-access` / an unconditional `gh pr merge` as defaults — that is what static scanners (Socket/Snyk) flag and what `automate.md` Step 3 forbids without explicit opt-in. Keep `confirm_privileged_mode` and its call before `launch_in_tmux_if_requested`: it lists the requested privileges and prompts `[y/N]`, is skipped by `--dry-run` and `--yes`, and **refuses** (rather than blocks) when no TTY is attached and `--yes` was not given. The tmux re-exec must propagate `--unattended`/`--auto-merge` and append `--yes`, so the human confirms once in the foreground and the detached child does not re-prompt. `--unattended` raises only the variable of the implementer CLI (`case "$IMPL_RUNNER"`: `claude` → `CLAUDE_PERMISSION_MODE=bypassPermissions`, `codex` → `CODEX_SANDBOX_MODE=danger-full-access`, any other runner dies instead of silently doing nothing), and `confirm_privileged_mode` names the implementer as the only widened role. Only the implementer runner reads these two variables; reviewer runners carry literal read-only flags (see *Reviewer runners*).
- **Detached runs should be first-class** — keep the `--tmux-session` / `--tmux-log` path working so long unattended batches can be launched safely without rewriting the script wrapper.
- **Stack-agnostic checks** — `run_checks` just iterates `CHECKS[]` from CLAUDE.md and runs each command via `bash -c "$cmd"` (a subshell, not `eval` in the current shell — same arbitrary-command support, better isolation, and it avoids the `eval` SAST flag). Do not add runtime/manifest guards (`package.json`, `pyproject.toml`, …); the commands themselves are the toolchain. Empty `CHECKS[]` is a valid no-op pass.
- **No assumed model CLIs beyond what roles need** — Sonnet/Opus/Codex are examples, not defaults. Generate runner functions from the user-confirmed model/CLI plan; for a single-review project set `REVIEW_B_ENABLED=false` (see *Single review is a toggle*).
- **Skill resolution is deterministic and logged.** `resolve_skill` runs exactly once per task (before implementation), seeded from `SKILL_MAP` (AGENTS.md *Skill Policy*). A `title:` regex (here and in `TEST_ELIGIBILITY`) is tested against the one string `"<title><newline><body>"`: `^` anchors the title start, `$` the end of the body, and `.` also matches the newline; to anchor on the title alone, match its prefix (`^docs:`) and do not end the pattern with `$`. An empty `SKILL_MAP` is a valid no-op (`(none)`); exactly one distinct match is chosen; **more than one distinct match is `(ambiguous)` — inject nothing and log it, never pick one**. There is no registry, no network, and no semantic fallback. An entry without `=`, with an empty skill name, of an unknown type, with an invalid `title:` regex, or a `label:` matcher that is dead (empty pattern, or `TASK_SOURCE_HAS_LABELS=false`) is warned and written to the log, never silently skipped or resolved to an empty skill. Every decision is written to `$logdir/skill-resolution.log` (`searched` / `candidates` / `chosen` / `warnings` / `reason`). The result is injected **only** into the implement/fix/refactor prompts — reviewers, check-fix, and the memory step stay skill-neutral.
- **Test policy is equally deterministic and opt-in.** `TEST_POLICY=off` (or absent/empty, normalized to `off`) is the valid default. `TEST_ELIGIBILITY` uses only explicit `label:` / `title:` include-or-except matchers, resolved once per task and logged to `$logdir/test-policy.log`. Resolution is fully deterministic with no "ambiguous" outcome: **`except` wins over `include`**; otherwise an `include` match is eligible; otherwise the base default follows the **DECLARED** matchers — allowlist (not eligible) whenever **any** `include` matcher is declared (a dead or typo include still counts as declared intent, so it can never flip the base to a denylist), denylist (eligible) only when every declared matcher is `except` and at least one of them is usable. A **dead** matcher (unknown type or effect, invalid regex, malformed entry, `label:` with an empty pattern, any `label:` matcher when `TASK_SOURCE_HAS_LABELS=false`) is logged as a warning and must **not** arm the denylist base. A set whose eligibility array is empty **or contains no usable matcher** is inert → `off` **and warned** (governance incompleteness, `[NEEDS GOVERNANCE]`) — it never falls through to "test every task". There is no heuristic "behavioral change" detector.
- **The gate is a red→green transition proof, not a green-only smoke test — but the RED *reason* is not exit-verified.** When the gate is available (policy enforced for the task **and** `TARGETED_TEST_CMD` set), a test-first sub-phase authors the test(s) and runs the target in `expect_red` mode **before** implementation — it must exit NON-ZERO (a tautological always-green test is rejected). **Honest scope:** a non-zero exit does *not* prove the failure is an assertion failure rather than a syntax/import/collection error — telling those apart needs framework-specific exit codes, which this stack-agnostic gate deliberately does not assume. "Fails for the right reason" is enforced by the **test-authoring prompt** (which demands an assertion-level failure, not a collection error) plus the post-impl rerun of the **same** target — not by parsing exit codes. Do not describe this as exit-verified. Once a RED target is confirmed, the script FREEZES that exact target for the rest of the task, so later prompts cannot retarget the proof to an easier test. The hard gate is armed (`TEST_GATE_ACTIVE`) **only under `required`** — there a confirmed RED makes the post-impl `expect_green` in `ensure_checks_pass` a blocking gate, and the pair proves the red→green *transition*. Under `preferred` the targeted test is still rerun after implementation, but an unresolved failure stays advisory rather than blocking the issue or triggering extra code mutation when `CHECKS[]` are already green. For `required`, an unprovable RED fails the issue; `preferred` degrades to advisory (no hard gate). The model writes exactly one affected test id/path to `TARGETED_TEST_FILE`; the script **sanitizes** it (allowlist `^[][A-Za-z0-9_./:@=+#-]+$`, and no leading `-` or `#`, which would read as an option or a comment) and substitutes it **shell-quoted** (`printf '%q'`) into `{TARGET}`, because that value is model-authored and reaches a shell (`bash -c`, never `eval`). `{TARGET}` must therefore stand unquoted in `TARGETED_TEST_CMD`; the script dies at startup on a quoted `{TARGET}`. In `expect_red`, exit 126 or 127 (not executable / command not found) is never accepted as RED: it logs "targeted test command not runnable", which fails the task under `required` and is advisory under `preferred`. If AGENTS.md says `required` but CLAUDE.md has no `TARGETED_TEST_CMD`, the script logs `[GOVERNANCE DRIFT]` and degrades to `preferred` instead of pretending a hard gate exists.
- **KNOWN LIMITATION — targeted gate, not a full no-regression gate (v1).** The red→green proof covers **only the one designated test**. It is deliberately *not* a general "no other test regressed" guarantee. Broad regression protection is exactly whatever `CHECKS[]` already runs and no more — so a project whose `CHECKS[]` is a full suite that is **already red before the task** will block the pipeline (every `CHECKS[]` command must pass), and a project whose `CHECKS[]` omits the suite gets no regression coverage beyond the single targeted test. Do not describe or generate this as a full no-regression gate. **Future extension (separate work):** capture a `CHECKS[]` baseline *only when the suite is green before the task*, compare after, and when the pre-task suite is already red, disable the no-regression comparison and log it clearly rather than blocking — see the matching note in `automate.md`.
