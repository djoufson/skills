# djoufson-skills

A personal collection of Claude Code skills distributed as a plugin marketplace.

## What's included

| Plugin | Description |
|--------|-------------|
| [`pr-review`](plugins/pr-review/) | Comprehensive PR code review with inline GitHub comments — covers code quality, security, performance, architecture, and testing |
| [`git-timesheet`](plugins/git-timesheet/) | Generate ERP-ready timesheet entries from git commit history using `git-report` |

## Installation

Add this collection to Claude Code by registering the marketplace manifest:

```bash
claude plugin add git@github.com:djoufson/skills.git
```

Once installed, skills become available as slash commands in any Claude Code session:

```
/pr-review 42
/git-timesheet
```

## Plugin structure

```
.claude-plugin/marketplace.json        # Marketplace manifest
plugins/<name>/plugin.json             # Plugin metadata
plugins/<name>/skills/<name>/SKILL.md  # Skill implementation
```

Each skill is a Markdown file with YAML frontmatter (name, description, arguments) and a body that contains the full prompt Claude Code executes when the skill is invoked.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
