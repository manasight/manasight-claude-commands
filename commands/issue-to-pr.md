---
description: Fix issue, create PR, iterate on review feedback, then report
argument-hint: <issue-url> [--plan] [--merge] [--opus]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(cargo:*), Bash(npm:*), Bash(npx:*), Bash(cd:*), Bash(sleep:*), Task, Read, Grep, Glob, Edit, Write, WebFetch
---

# Issue to PR

End-to-end: fix issue → create PR → iterate on feedback → report results.

## Arguments

- `$1`: Full GitHub issue URL (e.g., `https://github.com/your-org/your-repo/issues/5`). Bare numbers rejected — ambiguous in multi-repo workspaces.
- `--plan`: Pause after planning for user approval (default: proceed immediately)
- `--merge`: Auto-merge PR after approval (default: leave for manual review)
- `--opus`: Use Opus for Phase 1 implementation (default: Sonnet). Use for complex issues that need stronger reasoning.

```
/issue-to-pr https://github.com/your-org/your-repo/issues/5
/issue-to-pr https://github.com/your-org/another-repo/issues/42 --plan --merge
/issue-to-pr https://github.com/your-org/your-repo/issues/10 --opus
```

Arguments passed: `$ARGUMENTS`

## Current Context

- Today's date: !`date +%Y-%m-%d`
- Current branch: !`git rev-parse --abbrev-ref HEAD`
- Git status: !`git status --porcelain | head -5`
- Repository root: !`git rev-parse --show-toplevel`

## Pre-Commit Validation

<!-- CUSTOMIZE: If your CLAUDE.md doesn't have a "Pre-Commit Checklist" section,
     either add one or replace this with your specific commands
     (e.g., `npm test`, `cargo clippy && cargo test`, `pytest`). -->

**Repo-agnostic.** Read the repo's CLAUDE.md → find "Pre-Commit Checklist" section → execute every command listed. ALL must pass before committing. If no Pre-Commit Checklist section exists, STOP and ask the user what validation to run.

## Context Management (CRITICAL)

This workflow can run 10+ feedback iterations. To avoid exhausting context:

- **Phase 1 research** → Explore sub-agent (keeps search results out of parent context)
- **Phase 3 feedback iterations** → each iteration is a sub-agent (biggest savings)
- **Review comments** → truncate to 4000 chars; skip comments containing file contents (lock files, LICENSE, etc.)
- **Sub-agent models** → use `model: "sonnet"` for feedback iterations and CI polling; use `model: "opus"` for Phase 1 implementation when `--opus` is set, otherwise use default

## Phase 1: Fix Issue

### Parse Arguments

Extract from `$ARGUMENTS`: issue URL, `--plan`, `--merge`, `--opus`. If bare number, STOP and ask for full URL.

Parse URL `https://github.com/{OWNER}/{REPO}/issues/{NUMBER}` → OWNER, REPO, NUMBER, DISPLAY_REF (`{OWNER}/{REPO}#{NUMBER}`).

Determine current repo:
```bash
REPO_URL=$(git config --get remote.origin.url)
CURRENT_OWNER=$(echo $REPO_URL | sed -E 's/.*github\.com[:/]([^/]+)\/.*/\1/')
CURRENT_REPO=$(echo $REPO_URL | sed -E 's/.*\/([^/]+)(\.git)?$/\1/' | sed 's/\.git$//')
```

Close syntax: same repo → `Closes #{NUMBER}`, different repo → `Closes {OWNER}/{REPO}#{NUMBER}`.

### Load Repo Conventions

Read repo's CLAUDE.md. Extract: Pre-Commit Checklist, Testing Policy, Coding Conventions. Store for later phases.

### Research & Plan

1. Fetch issue: `gh issue view <NUMBER> --repo $OWNER/$REPO`
2. **Launch an Explore sub-agent** to research the codebase:
   > "Research codebase for issue [DISPLAY_REF]: [issue title]. Find: affected files, existing patterns/conventions, test structure, related code, edge cases. Report a concise summary with specific file paths and line references."
3. Using issue details + research summary, create implementation plan: summary, affected files, steps, test plan, risks

If `--plan`: present plan and STOP. Otherwise proceed immediately.

### Implement

1. Create branch from up-to-date main:
   ```bash
   git checkout main && git pull origin main
   git checkout -b issue/<NUMBER>-<brief-description>
   ```
2. **If `--opus`:** Launch a general-purpose sub-agent (`model: "opus"`) with the implementation plan, CLAUDE.md conventions, and affected file paths. The sub-agent implements the changes, adds/updates tests, and returns a summary of what was done. After the sub-agent completes, verify its changes in the parent context before proceeding.
   **Otherwise:** Implement directly in the parent context following CLAUDE.md conventions.
3. **Add/update tests (REQUIRED)** — follow repo's testing policy. If change falls under "What NOT to Test", document justification in PR body under `## Test Justification`

**Verify tests are added before proceeding.**

## Phase 2: Create PR

### Validate, Commit, Push

Run every command from the repo's Pre-Commit Checklist. **Do NOT commit if any fail.** Fix and re-run.

```bash
git add -u :/
git commit -m "Fix <DISPLAY_REF>: <description>

<detailed explanation>

Co-Authored-By: Claude <noreply@anthropic.com>"

git push -u origin issue/<NUMBER>-<brief-description>
```

New untracked files: review `git status` and `git add` individually. Do NOT blindly `git add :/`.

### Collect Coverage

<!-- CUSTOMIZE: Replace with your project's coverage tool, or remove if not applicable.
     Examples: cargo tarpaulin, pytest --cov, npx jest --coverage, go test -cover -->

After pre-commit validation passes, check CLAUDE.md for a coverage command and run it:

```bash
# Discover and run the repo's coverage tool, capture summary output
# Example (Rust):   cargo tarpaulin --all-features --ignore-tests 2>&1 | tail -1
# Example (Python): pytest --cov=src 2>&1 | tail -5
# Example (Node):   npx jest --coverage 2>&1 | tail -5
# Example (Go):     go test -cover ./... 2>&1 | tail -1
```

Store `COVERAGE_OUTPUT` for the PR body. If no coverage tool is configured, set `COVERAGE_OUTPUT` to "not configured".

### Create Pull Request

```bash
gh pr create --title "Fix <DISPLAY_REF>: <brief description>" --body "$(cat <<'EOF'
## Summary
- <bullet points>

## Changes Made
- <list of changes>

## Testing
- All tests passing (<TEST_COUNT> tests)
- Linting clean, formatted
- Code coverage: <COVERAGE_OUTPUT>

<CLOSES_SYNTAX>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Capture PR number.

## Phase 3: Feedback Loop

Max 10 iterations. **Each iteration runs as a sub-agent** to preserve parent context.

### Orchestration (parent does this)

<!-- CUSTOMIZE: This uses /self-review for automated code review. If you prefer
     external review tools or human-only review, replace the self-review step. -->

All repos use **self-review mode** — the `/self-review` command runs locally to generate structured review feedback with confidence scoring and false-positive filtering. No external CI auto-reviewer is needed.

For each iteration N (1..10):

1. **Launch a general-purpose sub-agent** (`model: "sonnet"`) with the prompt template below — substitute all `<VARIABLES>` before launching
2. Read the sub-agent's report
3. Based on reported status:
   - `approved` → exit loop, proceed to Phase 4
   - `feedback_addressed` → next iteration
   - `ci_unrecoverable` → ask user for guidance

### Self-Review Prompt

> You are addressing feedback for PR #`<PR_NUMBER>` in `<CURRENT_OWNER>`/`<CURRENT_REPO>` on branch `<BRANCH_NAME>`. This is iteration `<N>`.
>
> **Step 1 — CI:** Do NOT use `gh pr checks --watch` or while loops. Max 10 retries: run `gh pr checks <PR_NUMBER>`. If pending, `sleep 45` then retry. If failed: `gh run view <RUN_ID> --log-failed`, fix, validate, push. After 10 retries still pending → report `ci_pending`.
>
> **Step 2 — Code Review:** Run the `/self-review` skill on PR #`<PR_NUMBER>`. This runs 5 parallel review agents with confidence scoring (0-100) and filters issues below 75. It always posts a comment to the PR (even if no issues found), creating an audit trail. Treat all issues in the review comment as blocking — they have already passed the ≥75 confidence threshold.
>
> **Step 3 — Fix:** Read repo's CLAUDE.md for Pre-Commit Checklist. Fix all issues from the code review. Run ALL pre-commit validation commands. Then: `git add -u :/ && git commit -m "Address code review feedback (iteration <N>)` with summary and Co-Authored-By, then `git push`.
>
> **Step 4 — Re-review:** If fixes were made, run `/self-review` again on PR #`<PR_NUMBER>`. The command supports iterative re-review (no dedup blocking). If the review comment says "No issues found" → report `approved`. Otherwise → report `feedback_addressed` with unresolved items.
>
> **Report:** Return status (`approved`|`feedback_addressed`|`ci_failed`|`ci_pending`|`ci_unrecoverable`), 2-3 sentence summary, any unresolved items.

## Phase 4: Optional Merge

**Only if `--merge` flag.** `gh pr merge <PR_NUMBER> --squash --delete-branch`

## Phase 5: Report Results

Report: issue number/URL, PR number/URL, status (merged or ready for review), iterations count, next steps.

## Error Handling

- **CI unrecoverable** (3 failed attempts): list errors, ask user for guidance
- **Rebase conflicts**: attempt resolution; if unresolvable, ask user
- **Force pushes**: always use `--force-with-lease`
