# manasight-claude-commands

[![CI](https://github.com/manasight/manasight-claude-commands/actions/workflows/ci.yml/badge.svg)](https://github.com/manasight/manasight-claude-commands/actions/workflows/ci.yml)
[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue)](LICENSE-MIT)

Claude Code workflow commands for autonomous development. They turn Claude Code from an interactive assistant into an autonomous system that researches issues, writes code with tests, creates PRs, reviews its own work, and iterates on feedback — all without human intervention.

Built for and used daily on [Manasight](https://manasight.gg). Fork this repo and adapt the commands for your own project.

<!-- TODO: Read the full story: [How we built Manasight with autonomous Claude Code](blog-post-url) -->

## Commands

| Command | Description |
|---------|-------------|
| [`/issue-to-pr`](commands/issue-to-pr.md) | End-to-end: research issue, implement with tests, create PR, iterate on review feedback |
| [`/self-review`](commands/self-review.md) | Automated code review with 5 parallel agents and confidence scoring |
| [`/sequential-issues`](commands/sequential-issues.md) | Implement a chain of issues as stacked PRs with automated review |
| [`/merge-stack`](commands/merge-stack.md) | Squash-merge stacked PRs into main with automated rebasing |

## How they compose

```text
/issue-to-pr          The atomic unit. One issue in, one reviewed PR out.
    |
    +-- /self-review  The review gate. 5 parallel agents score findings
    |                 by confidence (0-100), filtering false positives.
    |
/sequential-issues    Chains /issue-to-pr across multiple issues.
    |                 Each PR stacks on the previous branch.
    |
    +-- /merge-stack  Collapses the stack. Retargets all PRs to main,
                      rebases unique commits, squash-merges in order.
```

`/issue-to-pr` works standalone for single issues. For batches, `/sequential-issues` orchestrates multiple `/issue-to-pr` runs with stacked branches, then `/merge-stack` collapses them cleanly into main.

## Installation

Copy the commands you want into your project's `.claude/commands/` directory:

```bash
# Clone this repo
git clone https://github.com/manasight/manasight-claude-commands.git

# Copy all commands
cp manasight-claude-commands/commands/*.md your-project/.claude/commands/

# Or copy individually
cp manasight-claude-commands/commands/issue-to-pr.md your-project/.claude/commands/
```

Then use them in Claude Code:

```text
/issue-to-pr https://github.com/your-org/your-repo/issues/42
/sequential-issues https://github.com/your-org/your-repo/issues/1 https://github.com/your-org/your-repo/issues/2 --merge
```

## What to customize

Each command has `<!-- CUSTOMIZE -->` comments marking the sections you should adapt. The key areas:

- **Pre-Commit Checklist** in your CLAUDE.md — the commands read this to know what validation to run before every commit (see [`examples/claude-md-snippet.md`](examples/claude-md-snippet.md))
- **Testing Policy** in your CLAUDE.md — what tests to write, what to skip
- **Coverage tool** — replace the multi-language examples with your specific tool
- **Confidence threshold** in `/self-review` — default is 75; lower catches more issues, higher reduces noise
- **Co-Authored-By** — update the commit trailer to match your preferred format

## Acknowledgments

`/self-review` is derived from Anthropic's [`/code-review`](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-review) plugin, licensed under Apache-2.0. Modifications include removing the dedup check (enabling re-review after fixes) and adding an "always comment" audit trail.

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT License](LICENSE-MIT) at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this project by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
