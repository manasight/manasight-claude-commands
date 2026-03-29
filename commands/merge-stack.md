---
description: Merge a stack of PRs (from /sequential-issues) into main via squash-merge with automated rebasing
argument-hint: <pr-numbers | pulls-url | epic-url> [--dry-run]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(sleep:*), AskUserQuestion, Read, Glob, Grep, WebFetch
---

# Merge Stack

Merges stacked PRs into main via squash-merge. Retargets all PRs to main first, then rebases each PR's unique commits with `--onto` after each squash-merge.

## Arguments

Input formats (one of):
1. **PR numbers** — in merge order: `101 102 103 104`
2. **Pulls page URL** — `https://github.com/owner/repo/pulls` — auto-discovers the stack
3. **Epic issue URL** — `https://github.com/owner/repo/issues/35` — finds PRs linked to child issues

Optional: `--dry-run` — show plan without merging.

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

1. Fetch epic body: `gh issue view {number} --repo {owner}/{repo} --json body`
2. Extract child issue refs from checkboxes (`- [ ] #123`, full URLs, `owner/repo#123`)
3. For each child, find open PRs: `gh pr list --search "closes #{number}" ...`
4. Walk base-branch chains (same as Format B)
5. Note child issues with no open PR

### Confirmation Gate

Print the discovered stack:

```
Discovered PR stack (merge order):

| #   | PR   | Branch                  | Base                    | Title                     |
|-----|------|-------------------------|-------------------------|---------------------------|
| 1   | #101 | issue/10-add-logging    | main                    | Add structured logging    |
| 2   | #102 | issue/11-retry-logic    | issue/10-add-logging    | Add retry logic           |
```

**STOP and ask user for confirmation.** If `--dry-run`, print table and stop.

## Algorithm

```
1. Retarget ALL PRs to main (BEFORE any merges)
2. Verify all PRs target main (hard stop if any don't)
3. Record old branch tips for --onto rebases
4. For each PR in order:
   a. Rebase (skip for first): git rebase --onto origin/main <old_tip> <branch>
   b. Verify PR targets main, then squash-merge with --delete-branch
```

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
git fetch origin main
```

Record `ORIGINAL_BRANCH` for restoration at the end.

### Step 3: Retarget All PRs to Main (CRITICAL)

**This step prevents orphaned PRs when branches are deleted. Do NOT skip it.**

For every PR whose base is not already `main`:
```bash
gh pr edit <number> --base main
```

**Verification (MANDATORY)** — re-fetch all PRs and confirm every base is `main`:
```bash
for pr in <PR_NUMBERS>; do
  base=$(gh pr view $pr --json baseRefName --jq '.baseRefName')
  if [ "$base" != "main" ]; then
    echo "FATAL: PR #$pr still targets $base, not main. Aborting."
    exit 1
  fi
done
```

**Do NOT proceed to Step 4 until all PRs target main.**

### Step 4: Record Old Tips

Before any rebasing, record every branch's current tip (needed for `--onto`):
```bash
for each PR:
  old_tip[pr] = $(git rev-parse origin/<branch>)
```

### Step 5: Merge Loop

For each PR in order:

#### 5a: Rebase (skip for first PR)

Replay only this PR's unique commits onto updated main:
```bash
git fetch origin main
git checkout <branch>
git rebase --onto origin/main <old_tip[previous_pr]>
git push --force-with-lease origin <branch>
```

If rebase conflicts: STOP, report which PR and files, ask user. Do NOT `--abort`.

#### 5b: Squash Merge

Pre-merge guard — verify base before every merge:
```bash
base=$(gh pr view <number> --json baseRefName --jq '.baseRefName')
if [ "$base" != "main" ]; then
  echo "ERROR: PR #<number> targets $base, not main. Retargeting..."
  gh pr edit <number> --base main
fi
```

Then merge:
```bash
sleep 3
gh pr merge <number> --squash --delete-branch
```

If "not mergeable": wait 5s, retry once. Still failing → report and stop.

Print: `Merged PR #<number>: <title>`

### Step 6: Restore & Report

```bash
git checkout main && git pull origin main
git stash pop 2>/dev/null
```

Print summary:
```
| # | PR    | Title                     | Status |
|---|-------|---------------------------|--------|
| 1 | #101  | Add structured logging    | Merged |
| 2 | #102  | Add retry logic           | Merged |
```

<!-- CUSTOMIZE: Adjust error handling, retry counts, and wait times
     to match your CI speed and merge queue behavior. -->

## Error Handling

- **No stack detected**: Report; suggest explicit PR numbers
- **Multiple stacks**: List each; ask user which to merge
- **Dirty working tree**: Stash automatically; restore on completion or failure
- **Rebase conflict**: Stop, print PR and files, ask user. Do NOT abort.
- **Merge fails**: Retry once after 5s. Still failing → stop and report.
- **PR not open**: Skip with warning
- **CI pending after rebase**: Wait up to 60s with retries
- **Partial completion**: Report which merged and which remain for re-run
