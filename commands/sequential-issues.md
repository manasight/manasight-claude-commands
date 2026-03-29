---
description: Implement a chain of GitHub issues using stacked branches with automated review iteration
argument-hint: <issue-urls...> [--plan] [--merge]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Bash(npm:*), Bash(npx:*), Bash(cd:*), Bash(sleep:*), Task, Read, Grep, Glob, Edit, Write, WebFetch
---

# Sequential Issues

Implements a chain of dependent GitHub issues using subagents with stacked branches. Each issue gets its own PR with automated review iteration.

## Arguments

- `issue-urls`: Space-separated full GitHub issue URLs. **Full URLs required** — bare numbers rejected (ambiguous in multi-repo workspaces).
- `--plan`: Subagents pause after planning for user approval (default: proceed immediately)
- `--merge`: Auto-merge all PRs in order after all are approved (default: leave for manual review)

Issues from different repos are supported for *reading*, but all PRs are created in the **current repository**.

```
/sequential-issues https://github.com/your-org/your-repo/issues/1 https://github.com/your-org/your-repo/issues/2
/sequential-issues https://github.com/your-org/other-repo/issues/10 https://github.com/your-org/your-repo/issues/13 --merge
```

Arguments passed: `$ARGUMENTS`

## Current Context

- Today's date: !`date +%Y-%m-%d`
- Current branch: !`git rev-parse --abbrev-ref HEAD`
- Git status: !`git status --porcelain | head -5`
- Repository: !`git config --get remote.origin.url | sed -E 's/.*github\.com[:/]([^/]+\/[^/]+?)(\.git)?$/\1/'`

## Stacked Branches

```
main
  └── issue/13-add-logging         ◄── PR #A targets main        (Closes #13)
        └── issue/14-retry-logic   ◄── PR #B targets 13's branch (Closes #14)
              └── issue/15-metrics ◄── PR #C targets 14's branch (Closes #15)
```

Each PR shows only its own changes. Before merging, retarget downstream PRs to main to prevent orphans.

## Orchestration

### Step 1: Parse Arguments

Extract issue URLs and flags from `$ARGUMENTS`. If any bare numbers, STOP and ask for full URLs.

For each URL, parse `https://github.com/{owner}/{repo}/issues/{number}` to get owner, repo, number.

Determine current repo:
```bash
REPO_URL=$(git config --get remote.origin.url)
CURRENT_OWNER=$(echo $REPO_URL | sed -E 's/.*github\.com[:/]([^/]+)\/.*/\1/')
CURRENT_REPO=$(echo $REPO_URL | sed -E 's/.*\/([^/]+)(\.git)?$/\1/' | sed 's/\.git$//')
```

For each issue, determine:
- `gh_view_cmd`: `gh issue view {number} --repo {owner}/{repo}`
- `closes_syntax`: same repo → `Closes #{number}`, different repo → `Closes {owner}/{repo}#{number}`
- `display_ref`: `{owner}/{repo}#{number}`

### Step 2: Initialize

Track: `previous_branch = "main"`, `branches_created = []`. Set plan_mode and merge_mode from flags.

### Step 3: Process Each Issue

For each issue IN ORDER:

1. Mark task `in_progress`
2. **Spawn subagent** (`subagent_type: "general-purpose"`) with the prompt below (substitute variables)
3. Capture branch name and PR number from subagent result
4. Verify: `gh pr view <PR> --json baseRefName,headRefName`
5. Set `previous_branch = new_branch`
6. Mark task `completed`

#### Subagent Prompt

```
You are implementing GitHub issue <DISPLAY_REF> as part of a stacked PR chain.

## Phase 1: Research & Plan

### Load Conventions
Read the repo's CLAUDE.md. Extract: Pre-Commit Checklist (exact commands to run before every commit), Testing Policy, Coding Conventions. If no Pre-Commit Checklist section exists, STOP and ask the user what validation to run.

### Understand Issue
Run: <GH_VIEW_CMD>
Extract requirements, acceptance criteria, constraints.

### Research & Plan
**Launch an Explore sub-agent** to research: "Research codebase for issue [DISPLAY_REF]: [issue title]. Find: affected files, patterns, test structure, edge cases. Report concise summary with file paths."
Using issue details + research summary, create implementation plan: summary, affected files, steps, test plan, risks.

<IF NOT PLAN MODE (default)>
Proceed immediately to Phase 2.
</IF NOT PLAN MODE>

<IF PLAN MODE>
Present plan and STOP. Wait for user input.
</IF PLAN MODE>

## Phase 2: Git Setup

```bash
git fetch origin
git checkout <PREVIOUS_BRANCH> && git pull origin <PREVIOUS_BRANCH>
git checkout -b issue/<NUMBER>-<brief-description>
```

## Phase 3: Implement

Follow your plan and CLAUDE.md conventions. **Add/update tests (REQUIRED)** — follow repo testing policy. If change falls under "What NOT to Test", document justification in PR body. Verify tests are added before proceeding.

## Phase 4: Validate, Commit, Push

Run every command from the repo's Pre-Commit Checklist. DO NOT commit if any fail. Fix and re-run.

```bash
git add -u :/
git commit -m "Fix <DISPLAY_REF>: <title>

<detailed explanation>

Co-Authored-By: Claude <noreply@anthropic.com>"
git push -u origin issue/<NUMBER>-<brief-description>
```

### Collect Coverage

<!-- CUSTOMIZE: Replace with your project's coverage tool, or remove if not applicable. -->

After pre-commit validation passes, check CLAUDE.md for a coverage command and run it:

```bash
# Discover and run the repo's coverage tool, capture summary output
# Example (Rust):   cargo tarpaulin --all-features --ignore-tests 2>&1 | tail -1
# Example (Python): pytest --cov=src 2>&1 | tail -5
# Example (Node):   npx jest --coverage 2>&1 | tail -5
# Example (Go):     go test -cover ./... 2>&1 | tail -1
```

Store `COVERAGE_OUTPUT` for the PR body. If no coverage tool is configured in CLAUDE.md, set `COVERAGE_OUTPUT` to "not configured".

## Phase 5: Create PR

```bash
gh pr create --base <PREVIOUS_BRANCH> --title "Fix <DISPLAY_REF>: <brief description>" --body "$(cat <<'PREOF'
## Summary
- <bullet points>

## Changes Made
- <list of changes>

## Testing
- All tests passing
- Linting clean, formatted
- Code coverage: <COVERAGE_OUTPUT>

## Stacked PR
Base: `<PREVIOUS_BRANCH>` — merge in order after earlier PRs.

<CLOSES_SYNTAX>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
PREOF
)"
```

Capture PR number.

## Phase 6: Feedback Loop

Two sequential gates — **both must pass** before an issue is considered approved.

<!-- CUSTOMIZE: This uses /self-review for automated review. If you prefer external
     review tools or human-only review, replace the self-review references. -->

### Gate 1: Code Review (max 10 iterations)

Each iteration runs as a nested sub-agent (`model: "sonnet"`, `subagent_type: "general-purpose"`) to preserve your context.

For each iteration N, launch a sub-agent with:
> You are addressing code review feedback for PR #`<PR_NUMBER>` in `<CURRENT_OWNER>`/`<CURRENT_REPO>` on branch `<BRANCH_NAME>`. Iteration `<N>`.
>
> **Code Review:** Run the `/self-review` skill on PR #`<PR_NUMBER>`. This runs 5 parallel review agents with confidence scoring (0-100) and filters issues below 75. It always posts a comment to the PR (even if no issues found), creating an audit trail. Treat all issues in the review comment as blocking — they have already passed the ≥75 confidence threshold.
>
> **Fix:** Read CLAUDE.md for Pre-Commit Checklist. Fix all issues from the code review. Run all validation, commit "Address code review feedback (iteration N)" with Co-Authored-By, push.
>
> **Re-review:** If fixes were made, run `/self-review` again on PR #`<PR_NUMBER>`. The command supports iterative re-review (no dedup blocking). If the review comment says "No issues found" → report `review_approved`. Otherwise → report `feedback_addressed` with unresolved items.
>
> **Report:** status (`review_approved`|`feedback_addressed`), summary, unresolved items.

Exit Gate 1 when status is `review_approved`.

### Gate 2: CI Green (mandatory, blocks "approved")

After Gate 1 passes, **wait for CI to complete and pass**. This is a hard gate — the issue is NOT approved until CI is green.

Launch a sub-agent (`model: "sonnet"`, `subagent_type: "general-purpose"`) with:
> You are verifying CI passes for PR #`<PR_NUMBER>` in `<CURRENT_OWNER>`/`<CURRENT_REPO>` on branch `<BRANCH_NAME>`.
>
> **Wait for CI:** Run `gh pr checks <PR_NUMBER> --repo <CURRENT_OWNER>/<CURRENT_REPO>`. If all checks pass → report `approved`. If checks are still pending, `sleep 45` and retry (max 20 retries = ~15 minutes). If a check fails:
>   1. Identify the failed run: `gh pr checks <PR_NUMBER> --repo <CURRENT_OWNER>/<CURRENT_REPO> --json name,state,link | jq '.[] | select(.state == "FAILURE")'`
>   2. Get the failure log: `gh run view <RUN_ID> --repo <CURRENT_OWNER>/<CURRENT_REPO> --log-failed 2>&1 | tail -80`
>   3. Diagnose and fix the root cause. Read CLAUDE.md for Pre-Commit Checklist. Run all local validation.
>   4. Commit fix: "Fix CI: <brief description>" with Co-Authored-By, push.
>   5. Wait for the new CI run to complete (restart the check loop).
>
> **Max 3 fix attempts.** If CI still fails after 3 fix+push cycles, report `ci_failed` with the failure summary.
>
> **Report:** status (`approved`|`ci_failed`|`ci_pending`), summary.

Exit Phase 6 when Gate 2 reports `approved`. DO NOT merge.

**Important:** If Gate 2 requires a fix+push, CI re-runs but self-review does NOT need to re-run (the fix is for CI infrastructure, not code quality). Only loop back to Gate 1 if the CI fix changes substantive application logic.

## Report Back

Return: branch name, PR number/URL, Gate 1 status (review), Gate 2 status (CI), overall status (approved/ci_failed/needs-review), brief summary.
```

### Step 4: Final Summary

After all issues processed, report stacked PRs table (order, issue, PR, base branch, status) and merge instructions:

To merge manually later: `/merge-stack <pr-numbers...>`

### Step 5: Auto-merge (only if --merge)

Invoke `/merge-stack <pr-numbers>` with the collected PR numbers in stack order. This handles retargeting, rebasing unique commits after each squash-merge, and branch cleanup. See `merge-stack.md` for the full algorithm.

## Error Handling

- **Subagent fails**: Ask user — Retry / Skip / Abort
- **Feedback loop timeout**: Continue to next issue; user addresses later
- **Merge conflict during merge-stack**: Handled by `/merge-stack` — stops and asks user
