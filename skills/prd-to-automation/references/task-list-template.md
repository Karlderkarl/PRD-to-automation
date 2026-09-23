# Task Source Template

The pipeline needs exactly one source of truth for what to work on. Choose one; wire only that one.

## Option A — GitHub Issues (default when a GitHub remote exists)

Convention the generated script relies on:

- **Eligibility**: issues carry a label (e.g., `agent:auto`). Create it once:
  ```bash
  gh label create agent:auto --description "Processed by auto-develop.sh" --color FBCA04
  ```
- **Dependencies**: put `Depends on #N` in the issue body. The script blocks the issue until every referenced issue is `CLOSED`.
- **Lifecycle**: the script branches `issue-<n>-<slug>`, implements, reviews, commits, opens a PR with `Closes #<n>`, and (only if opted in) merges.
- **Granularity**: one issue = one shippable unit, small enough for a meaningful single review.

**Seeding the backlog** — create one issue per phase sub-task from the AGENTS.md *Phase Plan*, with `Depends on #N` encoding the order. This is an outward-facing action: present the planned titles/bodies/dependencies and get explicit approval first, then run (per sub-task):

```bash
gh issue create \
  --title "<phase sub-task title>" \
  --label agent:auto \
  --body $'<one-line goal>\n\nDepends on #<N>'   # omit the Depends line if none
```

Concrete example:

```bash
gh issue create \
  --title "Phase 1 - foundation bootstrap" \
  --label agent:auto \
  --body $'Source: AGENTS.md Phase Plan\n\nGoal:\n- Bootstrap the repository foundation\n\nDepends on: none'

gh issue create \
  --title "Phase 2 - workflow automation" \
  --label agent:auto \
  --body $'Source: AGENTS.md Phase Plan\n\nGoal:\n- Add the automated workflow layer\n\nDepends on #1'
```

Because dependencies reference issue numbers that only exist after creation, seed in dependency order (roots first) and fill in each `Depends on #N` once the referenced issue number is known — or create the issues first, then `gh issue edit <n> --body` to add the `Depends on #N` lines. Capture the created numbers as you go.

## Option B — Local task-list file (default when there is no issue tracker)

A `refact-todo.md`-style file the script reads top-down for the next open task. Every task is a `### <n>. <title>` heading with exactly two field lines: `- depends on: none` or `- depends on: <n>[, <n>…]` (task numbers; a leading `#` is tolerated), and `- status: open` or `- status: done`. Only the script writes `status:`. Blueprint:

```markdown
# <Project> Task List

## Priority 1

### 1. <Task title>
<one-line goal>
- depends on: none
- status: open        <!-- open | done -->

### 2. <Task title>
<one-line goal>
- depends on: none
- status: open

## Priority 2

### 3. <Task title>
- depends on: 1, 2
- status: open

## Packages

| Package | Tasks | Effort |
|---|---|---|
| Minimal clean | 1-2 | 1-2 h |
| Sensible block | 1-3 | 4-6 h |
| Full pass | 1-N | 6-8 h |

## Post-package checks
After each package, run the project checks:
- `<check cmd 1>`
- `<check cmd 2>`
- quick visual check of the touched files/routes
```

How the script consumes it (replacing the `gh` selection in the template):

- Pick the first task whose `status: open` and whose `depends on` tasks are all `done` **on the base branch** (the script reads `git show "$BASE_BRANCH:$TASK_FILE"`; a flip on an unmerged task branch does not count). A missing or unparseable `depends on` line, or an unknown dependency, blocks the task (fail closed).
- Derive the branch from the task number/title with the AGENTS.md branch pattern; `task-<n>-<slug>` in the blueprint below is only the fallback.
- Set `TASK_SOURCE_HAS_LABELS=false`. For skill resolution, call `resolve_skill` with an empty label string (a task-list task has no labels): every `label:` matcher in the AGENTS.md *Skill Policy* is dead and warned at runtime, so only `title:` matchers can resolve a skill in this variant.
- **Wire the test-discipline contract identically to the issue variant** (it is task-source-general, not issue-only): call `resolve_test_policy` with an empty label string and **reset the full per-task test state at the top of every task exactly as the issue flow does** — `TARGETED_TEST_FILE` (set + `rm -f`), `FROZEN_TARGETED_TEST_TARGET=""`, and `TEST_GATE_ACTIVE=false` — then run the test-first **RED-before-GREEN** sub-phase before implementation when the task is test-eligible and a `TARGETED_TEST_CMD` exists. Resetting `FROZEN_TARGETED_TEST_TARGET` per task is **not optional**: a task-list run reuses the same process for many tasks in one invocation, so a RED-confirmed target left over from task N would otherwise be frozen into task N+1's GREEN reruns and "prove" a transition that never happened for that task. The same precedence and fail-safe rules carry through unchanged (`except` wins; allowlist base whenever an include is declared, denylist only for usable excepts; inert sets → `off`). Without this, test policy is documented but not actually generable for repos with no issue tracker.
  - **`label:` `TEST_ELIGIBILITY` matchers are dead here** (a task-list task has no labels), exactly like skill resolution — so only `title:` matchers make a task test-eligible. With `TASK_SOURCE_HAS_LABELS=false` the runtime warns every `label:` matcher and never lets it arm the denylist base, so a `label:`-only `except` set resolves to inert → `off` instead of testing *every* task. Still flag a `label:`-based `TEST_ELIGIBILITY` against a task-list source as `[GOVERNANCE DRIFT]` at generation/audit time and require `title:` matchers instead.
- After review passes and the memory step has archived the completed line, the **script** flips `status: open → done` (`task_mark_status`) before the final amend, so the flip is part of the task's one commit — the task-list file is the local analogue of "closing the issue". The model never edits `status:` (the memory prompt says so); nothing writes any other status, so there is no `doing`.
- The run stops at the commit: the task branch is kept for a human to review and merge into the base branch. A task branch that already carries own commits is finished work **awaiting review** and is skipped with a log (merge it, or delete the branch to redo the task) — not failed, not counted, exactly like the issue variant.
- A task with unmet dependencies is skipped with a log line, exactly like `Depends on #N`. Because the run stops at the commit (and the issue variant stops at the PR unless `--auto-merge`), a dependent task waits until its dependency is **merged** into the base branch: a batch run processes independent tasks only, and dependents follow in a later run.
- **`--dry-run` must stay read-only**: print the task it *would* run, but do **not** flip `status` or otherwise write the task file. A dry-run that dirties `refact-todo.md` would trip the next real run's clean-worktree guard. `task_mark_status` is guarded by the dry-run check (and the dry run returns before it is reached).

### Bash blueprint (local variant)

Generate the local variant from `auto-develop-template.md` and replace these five pieces; the logic of everything else (worktree guards, `refresh_base`, review loop, refactor pass, M6/M7 checks, memory step, main loop) stays as it is. Fill `TASK_SOURCE_HAS_LABELS=false`; `{{TASK_LABEL}}` is not used. `--issue <n>` selects task `<n>`. Then remove the issue residue outside the five pieces:

- drop `--auto-merge` (flag, usage line, `AUTO_MERGE`, its `confirm_privileged_mode` line): there is no PR to merge;
- drop `CODEX_SANDBOX_MODE` and its confirm line when no sandboxed reviewer CLI is selected;
- reword the usage text, the checkpoint commit message (`feat: implement #$issue`), and the `#$issue` / `gh` wording in logs to "task".

Validate with `--dry-run` before the first commit: piece 5 then reads the working-tree copy of the task file and says so.

1. Replace `issue_has_label` and `check_dependencies` (section *Issue eligibility + dependency blocking*):

```bash
TASK_FILE="{{TASK_FILE}}"   # e.g. refact-todo.md

# The status of record is TASK_FILE as committed on BASE_BRANCH; a flip on an unmerged task
# branch does not count. task_index prints one TSV line per task: <n>\t<status>\t<depends on>\t<title>.
# A task runs from "### <n>. <title>" to the next heading; CRLF is tolerated. A missing field
# reads "(missing)": the task is then never selected (status) or blocked (depends on).
# Exception: a dry run before TASK_FILE is committed (automate Step 6 validates before the user
# commits) reads the working-tree copy; piece 5 detects that once and says so. A real run never does.
TASK_SOURCE_WORKTREE=false
task_source() {
  if [[ "$TASK_SOURCE_WORKTREE" == true ]]; then cat "$TASK_FILE"; else git show "$BASE_BRANCH:$TASK_FILE"; fi; }
task_index() {
  local src; src="$(task_source)" || return 1
  awk '
    function flush() { if (n != "") printf "%s\t%s\t%s\t%s\n", n, st, dp, ti }
    { sub(/\r$/, "") }
    /^### [0-9]+\. / { flush(); n = $2; sub(/\.$/, "", n); ti = $0; sub(/^### [0-9]+\. /, "", ti)
                       st = "(missing)"; dp = "(missing)"; next }
    /^#/             { flush(); n = ""; next }
    n != "" && /^- status:/     { st = $0; sub(/^- status:[[:space:]]*/, "", st); sub(/[[:space:]]*(<!--.*)?$/, "", st) }
    n != "" && /^- depends on:/ { dp = $0; sub(/^- depends on:[[:space:]]*/, "", dp); sub(/[[:space:]]+$/, "", dp) }
    END { flush() }' <<< "$src"; }
task_field() {  # <n> <column: 2=status 3=depends on 4=title> -> empty when the task does not exist
  local idx; idx="$(task_index)" || return 1
  awk -F'\t' -v n="$1" -v c="$2" '$1 == n && !seen { print $c; seen = 1 }' <<< "$idx"; }
task_body() {  # <n> -> the task's description lines (headings and field lines excluded)
  local src; src="$(task_source)" || return 1
  awk -v want="$1" '
    { sub(/\r$/, "") }
    /^### [0-9]+\. / { cur = $2; sub(/\.$/, "", cur); next }
    /^#/             { cur = ""; next }
    cur == want && !/^- (status|depends on):/ && NF { print }' <<< "$src"; }

# M5 for the task list: every task number on the "depends on" line must be `done` on
# BASE_BRANCH. Fails CLOSED: an unreadable file, a missing or unparseable line, or an unknown
# dependency blocks the task.
check_dependencies() {  # <n>
  local deps nums d st
  deps="$(task_field "$1" 3)" || { log "Cannot read $TASK_FILE on $BASE_BRANCH; treating task $1 as blocked."; return 1; }
  [[ "${deps,,}" == none ]] && return 0
  nums="$(grep -oE '[0-9]+' <<< "$deps" || true)"
  [[ -n "$nums" ]] || { log "Task $1: unparseable 'depends on: $deps'; treating it as blocked."; return 1; }
  for d in $nums; do
    st="$(task_field "$d" 2)" || { log "Cannot read $TASK_FILE on $BASE_BRANCH; treating task $1 as blocked."; return 1; }
    [[ "$st" == "done" ]] || { log "Blocked by task $d (status on $BASE_BRANCH: ${st:-unknown task})"; return 1; }
  done; return 0; }

# Script-owned status flip (contract M7: the task source's status is a pipeline write; the model
# never edits it). Called after the memory step and before the final amend, so the flip lands in
# the task's one commit. A dry run never writes. Pure bash, so every other line stays byte-identical
# (some Windows awk builds drop CR on input); fails without a status line.
task_mark_status() {  # <n> <status>
  [[ "$DRY_RUN" != true ]] || { log "[dry-run] would set task $1 status: $2"; return 0; }
  local tmp line bare cur="" flipped=false re='^### ([0-9]+)\. '
  tmp="$(mktemp)" || return 1
  while IFS= read -r line || [[ -n "$line" ]]; do
    bare="${line%$'\r'}"
    if [[ "$bare" =~ $re ]]; then cur="${BASH_REMATCH[1]}"
    elif [[ "$bare" == "#"* ]]; then cur=""
    elif [[ "$cur" == "$1" && "$flipped" == false && "$bare" == "- status:"* ]]; then
      line="- status: $2${line#"$bare"}"; flipped=true   # keeps a trailing CR
    fi
    printf '%s\n' "$line"
  done < "$TASK_FILE" > "$tmp" || { rm -f "$tmp"; log "ERROR: cannot rewrite $TASK_FILE"; return 1; }
  [[ "$flipped" == true ]] || { rm -f "$tmp"; log "ERROR: no status line for task $1 in $TASK_FILE"; return 1; }
  cat "$tmp" > "$TASK_FILE" && rm -f "$tmp"; }
```

2. Replace `branch_awaiting_review` (no PRs in this variant):

```bash
# A task branch with own commits ahead of BASE_BRANCH is finished work awaiting review/merge
# (or an earlier run failed after the checkpoint): skip it, never reprocess it on top of itself.
branch_awaiting_review() {  # <branch> -> 0 awaiting (AWAIT_REASON set), 1 free, 2 cannot tell
  local own; AWAIT_REASON=""
  git show-ref --verify --quiet "refs/heads/$1" || return 1
  own="$(git rev-list --count "$BASE_BRANCH..$1")" || return 2
  [[ "$own" -gt 0 ]] || return 1
  AWAIT_REASON="branch $1 has $own own commit(s) ahead of $BASE_BRANCH"; return 0; }
```

3. In `process_issue`, replace the task-source reads (from `title=` through `branch=`):

```bash
  title="$(task_field "$issue" 4)" || { log "ERROR: cannot read task $issue from $TASK_FILE on $BASE_BRANCH"; return 1; }
  [[ -n "$title" ]]                || { log "ERROR: task $issue not found in $TASK_FILE"; return 1; }
  body="$(task_body "$issue")"     || { log "ERROR: cannot read task $issue from $TASK_FILE on $BASE_BRANCH"; return 1; }
  labels=""                        # label-less source (TASK_SOURCE_HAS_LABELS=false)
  branch="task-$issue-$(slugify "$title")"   # fallback; use the AGENTS.md branch pattern
```

4. In `process_issue`, replace step 5 (from `# 5. Fold memory` to the end of the function; no push, no PR, no merge):

```bash
  # 4a. Script-owned status flip, after the memory step and before the final amend.
  task_mark_status "$issue" "done" \
    || { log "ERROR: could not mark task $issue done; checkpoint kept on $branch."; return_to_base; return 1; }

  # 5. Fold memory + status + the final message into the ONE task commit, then stop at the
  #    commit: a human reviews the branch and merges it into BASE_BRANCH.
  stage_repo_changes
  local refactor_summary="off"
  [[ "$REFACTOR" == true ]] && refactor_summary="on (${REFACTOR_ROUNDS} round(s) applied)"
  git commit --amend -m "feat: implement task $issue - $title

Automated via auto-develop.sh. Model plan: $IMPL_LABEL | $REVIEW_A_LABEL${REVIEW_B_LABEL:+, $REVIEW_B_LABEL}
Correctness review rounds: $review_rounds/$MAX_ROUNDS (delivered A/B rounds incl. accepted refactor re-reviews: $DELIVERED_REVIEW_ROUNDS)
Refactor pass: $refactor_summary
Task: $TASK_FILE #$issue" >/dev/null \
    || { log "ERROR: final commit --amend failed for task $issue; checkpoint kept on $branch."; return_to_base; return 1; }
  git checkout "$BASE_BRANCH" >/dev/null 2>&1 || log "WARN: could not return to $BASE_BRANCH"
  log "Task $issue committed on $branch; awaiting human review and merge into $BASE_BRANCH."
  log "=== Done task $issue ==="; }
```

5. Replace the candidate selection (the `CANDIDATES=()` block before the main loop):

```bash
if [[ "$DRY_RUN" == true ]] && ! git cat-file -e "$BASE_BRANCH:$TASK_FILE" 2>/dev/null && [[ -f "$TASK_FILE" ]]; then
  TASK_SOURCE_WORKTREE=true
  log "[dry-run] NOTE: $TASK_FILE is not committed on $BASE_BRANCH; reading the working-tree copy. A real run refuses until it is committed."
fi
CANDIDATES=()
if [[ -n "$TARGET_ISSUE" ]]; then   # --issue <n> selects task <n>
  st="$(task_field "$TARGET_ISSUE" 2)" || die "Cannot read $TASK_FILE on $BASE_BRANCH."
  [[ "$st" == open ]] || die "Task $TARGET_ISSUE is '${st:-not found}', not open."
  CANDIDATES=("$TARGET_ISSUE")
else
  idx="$(task_index)" || die "Cannot read $TASK_FILE on $BASE_BRANCH."
  mapfile -t CANDIDATES < <(awk -F'\t' '$2 == "open" { print $1 }' <<< "$idx")
fi
```

## Option C — MEMORY.md "Next Up" (minimal projects only)

For very small projects, the script can read the single top item from MEMORY.md "Next Up". No labels, no PRs — implement, check, review, commit on a branch, advance the line. Use only when neither issues nor a task-list file is warranted.

- **Test discipline carries identically here too** (it is task-source-general, never issue-only): if governance opted in, wire `resolve_test_policy` with an empty label string, the **full per-task reset** of `TARGETED_TEST_FILE` / `FROZEN_TARGETED_TEST_TARGET` / `TEST_GATE_ACTIVE`, and the RED-before-GREEN sub-phase — exactly as Option B above spells out (and as `automate.md` / `extraction-checklist.md` require for *any* source). Like the task-list variant, a MEMORY.md "Next Up" item has **no labels**: set `TASK_SOURCE_HAS_LABELS=false`, so every `label:` matcher is dead and warned at runtime and only `title:` `TEST_ELIGIBILITY` matchers can make it test-eligible; flag a `label:`-based `TEST_ELIGIBILITY` against this source as `[GOVERNANCE DRIFT]`. Where test policy is `off` (the default for minimal projects), none of this emits — the minimal flow above is the whole script.

## Guidelines

- Never wire two sources at once — pick A, B, or C with the user (`[USER DECISION REQUIRED]` if unclear).
- Keep dependency semantics identical across sources: a blocked task is skipped, not failed.
- The task source defines *what* to build; the governance defines *how*. Do not let the task file restate coding standards or security rules — those live in SOUL.md/AGENTS.md.
- In GitHub-issue mode, "seed the backlog" means actual `gh issue create` commands, not just creating the label.
