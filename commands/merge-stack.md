---
description: Merge a stack of PRs (from /sequential-issues) into main via squash-merge with automated rebasing
argument-hint: <pr-numbers | pulls-url | epic-url> [--no-branch] [--no-review] [--dry-run]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(sleep:*), AskUserQuestion, Task, Read, Glob, Grep, WebFetch
---

# Merge Stack

Collapses a stack of PRs into a **single integration PR** for review and merge.

**Default mode (recommended):** create a fresh integration branch off `origin/main`, retarget every PR in the stack onto that branch, squash-merge each PR into the integration branch in order, then open one PR (integration → `main`) and run `/self-review` on it. This gives reviewers one cohesive change to look at instead of N noisy squash commits on `main`, and gives `/self-review` a chance to catch issues that only emerge when the full stack lands together.

**Legacy mode (`--no-branch`):** retarget every PR to `main` and squash-merge each in order, exactly as before.

## Arguments

Input formats (one of):
1. **PR numbers** — in merge order: `101 102 103 104`
2. **Pulls page URL** — `https://github.com/owner/repo/pulls` — auto-discovers the stack
3. **Epic issue URL** — `https://github.com/owner/repo/issues/35` — finds PRs linked to child issues

Flags:
- `--no-branch` — skip the integration branch entirely. Retarget and squash-merge each PR directly to `main` (legacy behavior).
- `--no-review` — default flow, but skip the `/self-review` step after opening the integration PR. The integration PR is created and left for the user to review and merge.
- `--dry-run` — show plan without merging or creating branches.

`--no-branch` and `--no-review` are mutually exclusive (`--no-review` is meaningless without an integration PR).

Arguments passed: `$ARGUMENTS`

## Current Context

- Today's date: !`date +%Y-%m-%d`
- Current branch: !`git rev-parse --abbrev-ref HEAD`
- Git status: !`git status --porcelain | head -5`
- Repository: !`git config --get remote.origin.url | sed -E 's/.*github\.com[:/]([^/]+\/[^/]+?)(\.git)?$/\1/'`

## Step 0: Discover Stack

Parse `$ARGUMENTS` to determine format, then build an ordered PR list.

### Format A: Explicit PR Numbers

Extract numbers directly. Validate each is open.

### Format B: Pulls Page URL

Matches: `https://github.com/{owner}/{repo}/pulls`

```bash
gh pr list --repo {owner}/{repo} --state open --json number,title,headRefName,baseRefName
```

Walk the base-branch chain to find the stack (`main ← PR#1 ← PR#2 ← PR#3`):
1. Map `head_branch → PR` and `base_branch → [PRs]`
2. Find root (base=`main`, head is another PR's base)
3. Walk root→leaf to get merge order
4. Multiple stacks → ask user which one. No stack → report and stop.

### Format C: Epic Issue URL

Matches: `https://github.com/{owner}/{repo}/issues/{number}`

1. Fetch epic body: `gh issue view {number} --repo {owner}/{repo} --json body,title`
2. Extract child issue refs from checkboxes (`- [ ] #123`, full URLs, `owner/repo#123`)
3. For each child, find open PRs: `gh pr list --search "closes #{number}" ...`
4. Walk base-branch chains (same as Format B)
5. Note child issues with no open PR
6. Capture epic number + title for branch naming and PR title

### Determine Mode and Integration Branch Name

Parse flags. Mode is one of:
- `default` — integration branch + integration PR + self-review
- `--no-review` — integration branch + integration PR, no self-review
- `--no-branch` — direct-to-main legacy mode

For `default` and `--no-review` modes, derive an integration branch name:
- **Format C (epic):** `merge-stack/issue-<epic-num>-<short-slug>`, where `<short-slug>` is the epic title lowercased, non-alphanumerics replaced with `-`, truncated to ~30 chars
- **Otherwise:** `merge-stack/<YYYY-MM-DD>-prs-<first>-<last>` (e.g., `merge-stack/2025-04-25-prs-101-104`)

If a branch with this name already exists locally or on `origin`, append `-2`, `-3`, etc.

### Confirmation Gate

Print the discovered stack and the plan:

```
Discovered PR stack (merge order):

| #   | PR   | Branch                  | Base                    | Title                     |
|-----|------|-------------------------|-------------------------|---------------------------|
| 1   | #101 | issue/10-add-logging    | main                    | Add structured logging    |
| 2   | #102 | issue/11-retry-logic    | issue/10-add-logging    | Add retry logic           |

Mode: default (integration branch + self-review)
Integration branch: merge-stack/issue-35-roadmap-rollup
Final step: open PR (merge-stack/issue-35-roadmap-rollup → main) and run /self-review
```

**STOP and ask user for confirmation.** Allow the user to override the integration branch name. If `--dry-run`, print and stop.

## Algorithm

### Default mode (with integration branch)

```
1. Stash, fetch origin
2. Create integration branch from origin/main; push to origin
3. Retarget ALL PRs (including the tip) to the integration branch
4. Verify all PRs target the integration branch (hard stop if any don't)
5. Record old branch tips for --onto rebases
6. For each PR in order:
   a. Rebase (skip first): git rebase --onto origin/<integration> <old_tip> <branch>
   b. Verify base, then squash-merge with --delete-branch into integration branch
7. Pull the integration branch
8. Open integration PR (integration → main) summarizing every merged PR
9. (Default only) Spawn sub-agent to run /self-review on the integration PR
```

### Legacy mode (`--no-branch`)

```
1. Stash, fetch origin
2. Retarget ALL PRs to main
3. Verify all PRs target main
4. Record old branch tips
5. For each PR in order:
   a. Rebase (skip first): git rebase --onto origin/main <old_tip> <branch>
   b. Verify base, then squash-merge with --delete-branch into main
6. Pull main, restore stash
```

In both modes, the rebase/merge loop is identical — the only difference is the target branch (`<integration>` vs `main`) and the post-merge step (open integration PR vs nothing).

`<TARGET>` below means the integration branch in default/`--no-review` modes, or `main` in `--no-branch` mode.

## Execution

### Step 1: Validate

```bash
for pr in <PR_NUMBERS>; do
  gh pr view $pr --json number,title,headRefName,baseRefName,state
done
```

### Step 2: Stash & Fetch

```bash
git stash --include-untracked -m "merge-stack: stashing before merge"
git fetch origin
```

Record `ORIGINAL_BRANCH` for restoration at the end.

### Step 3: Create Integration Branch (skip in `--no-branch` mode)

```bash
git checkout -b <INTEGRATION_BRANCH> origin/main
git push -u origin <INTEGRATION_BRANCH>
```

If push fails because the branch already exists on `origin`, the discovery step should have caught this and chosen a different name. Hard stop if it slipped through.

### Step 4: Retarget All PRs to `<TARGET>` (CRITICAL)

**This step prevents orphaned PRs when branches are deleted. Do NOT skip it.**

For every PR whose base is not already `<TARGET>`:
```bash
gh pr edit <number> --base <TARGET>
```

**Verification (MANDATORY)** — re-fetch all PRs and confirm every base is `<TARGET>`:
```bash
for pr in <PR_NUMBERS>; do
  base=$(gh pr view $pr --json baseRefName --jq '.baseRefName')
  if [ "$base" != "<TARGET>" ]; then
    echo "FATAL: PR #$pr still targets $base, not <TARGET>. Aborting."
    exit 1
  fi
done
```

**Do NOT proceed to Step 5 until all PRs target `<TARGET>`.**

### Step 5: Record Old Tips

Before any rebasing, record every branch's current tip (needed for `--onto`):
```bash
for each PR:
  old_tip[pr] = $(git rev-parse origin/<branch>)
```

### Step 6: Merge Loop

For each PR in order:

#### 6a: Rebase (skip for first PR)

Replay only this PR's unique commits onto the updated `<TARGET>`:
```bash
git fetch origin <TARGET>
git checkout <branch>
git rebase --onto origin/<TARGET> <old_tip[previous_pr]>
git push --force-with-lease origin <branch>
```

If rebase conflicts: STOP, report which PR and files, ask user. Do NOT `--abort`.

#### 6b: Squash Merge

Pre-merge guard — verify base before every merge:
```bash
base=$(gh pr view <number> --json baseRefName --jq '.baseRefName')
if [ "$base" != "<TARGET>" ]; then
  echo "ERROR: PR #<number> targets $base, not <TARGET>. Retargeting..."
  gh pr edit <number> --base <TARGET>
fi
```

Then merge:
```bash
sleep 3
gh pr merge <number> --squash --delete-branch
```

If "not mergeable": wait 5s, retry once. Still failing → report and stop.

Print: `Merged PR #<number>: <title>`

### Step 7: Restore Working Tree

```bash
git checkout <TARGET> && git pull origin <TARGET>
git stash pop 2>/dev/null
```

In `--no-branch` mode, this is the final step. Print the summary table and exit.

### Step 8: Open Integration PR (default and `--no-review` modes)

Build the body from the merged PRs. For Format C (epic), title the PR after the epic; otherwise use a generic title.

```bash
gh pr create \
  --base main \
  --head <INTEGRATION_BRANCH> \
  --title "<INTEGRATION_PR_TITLE>" \
  --body "$(cat <<'PREOF'
## Summary

Integration PR for the following stack of changes:

| # | PR    | Title                     |
|---|-------|---------------------------|
| 1 | #101  | Add structured logging    |
| 2 | #102  | Add retry logic           |

<!-- If sourced from an epic, link it: Closes owner/repo#<epic-num> -->

## Review

This PR collapses N stacked PRs into a single review. Each commit on this branch is the squash-merge of one upstream PR; see the table above for individual scope. `/self-review` results posted as a comment.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
PREOF
)"
```

Capture `INTEGRATION_PR_NUMBER` from the create output.

### Step 9: Run `/self-review` (default mode only — skip if `--no-review`)

Spawn a sub-agent to run `/self-review` on the integration PR. This isolates the review from the orchestrator's context (`/self-review` itself fans out 5 parallel agents).

Use the `Task` tool with `subagent_type: "general-purpose"` and a prompt like:

> Run the `/self-review` skill on PR #`<INTEGRATION_PR_NUMBER>` in `<OWNER>/<REPO>`. Report when the review comment has been posted (or "no issues found" if clean). Do not attempt to fix anything — just run the review and report.

Wait for the sub-agent to return. Surface its summary in the final report.

### Step 10: Final Report

```
| # | PR    | Title                     | Status |
|---|-------|---------------------------|--------|
| 1 | #101  | Add structured logging    | Merged |
| 2 | #102  | Add retry logic           | Merged |

Integration PR: #<INTEGRATION_PR_NUMBER> (<INTEGRATION_BRANCH> → main)
Self-review: <posted | skipped (--no-review) | N/A (--no-branch)>
Next step: review the integration PR and squash-merge it into main when ready.
```

In `--no-branch` mode, omit the integration PR / self-review lines.

<!-- CUSTOMIZE: Adjust error handling, retry counts, and wait times
     to match your CI speed and merge queue behavior. -->

## Error Handling

- **No stack detected**: Report; suggest explicit PR numbers
- **Multiple stacks**: List each; ask user which to merge
- **Dirty working tree**: Stash automatically; restore on completion or failure
- **Integration branch name collides**: Append `-2`, `-3`, etc. and retry once. Still failing → ask user for a name.
- **Rebase conflict**: Stop, print PR and files, ask user. Do NOT abort.
- **Merge fails**: Retry once after 5s. Still failing → stop and report.
- **PR not open**: Skip with warning
- **CI pending after rebase**: Wait up to 60s with retries
- **`gh pr create` fails for the integration PR**: Report the error and the integration branch name. The merges already landed on the branch — the user can open the PR manually with `gh pr create --base main --head <INTEGRATION_BRANCH>`.
- **Self-review sub-agent fails**: Report failure with the integration PR URL. The user can run `/self-review <PR>` manually.
- **Both `--no-branch` and `--no-review` provided**: Stop with an error explaining they're mutually exclusive.
- **Partial completion**: Report which merged and which remain for re-run
