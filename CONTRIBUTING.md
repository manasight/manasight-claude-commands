# Contributing to manasight-claude-commands

Contributions are welcome! Whether it's improving an existing command, proposing a new one, or fixing a bug, we appreciate your help.

## Reporting Issues

Open a [GitHub issue](https://github.com/manasight/manasight-claude-commands/issues) with:

- A clear description of the problem or suggestion
- Which command is affected
- Steps to reproduce (if applicable)

## Suggesting Command Improvements

Use the `command-improvement` label. Describe:

- Which command you're improving
- What the current behavior is
- What you'd like it to do instead
- Why the change is useful beyond your specific project

## Proposing New Commands

Use the `new-command` label. Describe:

- The workflow the command automates
- Inputs and outputs
- How it composes with the existing commands (if applicable)

## Submitting Pull Requests

1. Fork the repository and create a feature branch
2. Make your changes
3. Ensure markdownlint passes (see below)
4. Open a pull request against `main`

## Development Setup

```bash
git clone https://github.com/manasight/manasight-claude-commands.git
cd manasight-claude-commands

# Lint markdown
npx markdownlint-cli2 "commands/*.md" "*.md"
```

## Testing Changes

Copy modified command files to a project's `.claude/commands/` directory and run them with Claude Code. There are no automated tests beyond markdownlint — the commands are tested by using them.

## Licensing of Contributions

By submitting a pull request, you agree that your contributions are licensed under the same terms as the project: [MIT](LICENSE-MIT) OR [Apache-2.0](LICENSE-APACHE), at the user's option.

## Code of Conduct

This project follows the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).
