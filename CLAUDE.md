# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of Claude Code skills distributed as a plugin marketplace. Users install the whole collection via the `.claude-plugin/marketplace.json` manifest, which registers each plugin so its skills become available as slash commands in Claude Code.

## Repository structure

```
.claude-plugin/marketplace.json        # Top-level marketplace manifest (registry of all plugins)
plugins/<name>/plugin.json             # Plugin metadata (name, version, author, description)
plugins/<name>/skills/<skill>/SKILL.md # Skill implementation — frontmatter + prompt instructions
```

Each skill is a Markdown file with YAML frontmatter. The frontmatter declares the skill's `name`, `description` (used for trigger matching), and optionally `user_invocable: true` with typed `arguments`. The body is the prompt Claude Code receives when the skill fires.

## Adding a new plugin

1. Create `plugins/<name>/plugin.json` with `name`, `version`, `description`, and `author`.
2. Create `plugins/<name>/skills/<name>/SKILL.md` with frontmatter + instructions.
3. Register it in `.claude-plugin/marketplace.json` under the `plugins` array — include `name`, `source` (relative path), `description`, `version`, `author`, and `category`.

No build step, no install command, no test runner — the repo is pure JSON + Markdown.

## Skill authoring conventions

- **`description`** in the frontmatter is the trigger surface: write it to match the natural language the user would type, not just a label.
- **`user_invocable: true`** makes the skill callable as a slash command; omit it for skills that trigger automatically.
- **Arguments** declared in frontmatter use `{{arg}}` interpolation in the skill body.
- Skills should be self-contained: include every bash command, flag, and decision rule needed so Claude can execute without context from outside the skill file.
- Structure skills as numbered steps — each step should have a clear input and output so Claude knows when it's done.
