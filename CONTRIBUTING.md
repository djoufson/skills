# Contributing

## Adding a new plugin

1. Create the plugin directory and metadata:

```bash
mkdir -p plugins/<name>/.claude-plugin
mkdir -p plugins/<name>/skills/<name>
```

```json
// plugins/<name>/.claude-plugin/plugin.json
{
  "name": "<name>",
  "version": "1.0.0",
  "description": "...",
  "author": {
    "name": "Your Name",
    "email": "you@example.com"
  }
}
```

2. Write the skill in `plugins/<name>/skills/<name>/SKILL.md`:

```markdown
---
name: <name>
description: <trigger description — written to match natural language the user would type>
user_invocable: true
arguments:
  - name: arg
    description: "..."
    required: true
---

# Skill Title

Step-by-step instructions for Claude to execute...
```

3. Register it in `.claude-plugin/marketplace.json` under `plugins`:

```json
{
  "name": "<name>",
  "source": "./plugins/<name>",
  "description": "...",
  "version": "1.0.0",
  "author": { "name": "Your Name" },
  "category": "development"
}
```

## Skill authoring tips

- The `description` YAML header field is the trigger surface — write it in natural language, as a user would phrase the request.
- Skills must be self-contained: include every command, flag, and decision rule so Claude can execute without external context.
- Structure as numbered steps with clear inputs and outputs.
- Use `{{arg}}` syntax for argument interpolation in the skill body.

## Submitting a PR

Open a pull request against `main`. The `/pr-review` skill will run automatically to provide feedback.
