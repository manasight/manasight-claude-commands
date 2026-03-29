# manasight-claude-commands

## Purpose

Markdown command files for Claude Code. Each file in `commands/` defines an autonomous workflow.

## Conventions

- Follow existing command file structure (YAML frontmatter, sections, formatting)
- Use `<!-- CUSTOMIZE: ... -->` comments for user-configurable sections
- Test changes by copying to a project's `.claude/commands/` directory and running
- markdownlint must pass: `npx markdownlint-cli2 "commands/*.md" "*.md"`
