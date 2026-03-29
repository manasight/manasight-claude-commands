# Example CLAUDE.md Snippet

The commands in this repo read your project's `CLAUDE.md` to discover conventions and validation steps. Here's an example of the sections they look for.

---

## Pre-Commit Checklist (CRITICAL)

<!-- The /issue-to-pr and /sequential-issues commands read this section
     and run every command listed before committing. -->

- [ ] Tests pass: `npm test`
- [ ] Lint clean: `npm run lint`
- [ ] Format verified: `npm run format:check`
- [ ] New/updated tests for every code change
- [ ] All files staged: `git add :/ && git status`

## Testing Policy

<!-- The commands reference this section to decide what tests to write
     and when to skip testing. -->

- Unit tests for all business logic
- Integration tests for API endpoints
- **What NOT to Test:** generated types, third-party library wrappers, configuration-only changes
