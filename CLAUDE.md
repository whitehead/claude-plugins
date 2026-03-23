# Claude Code Instructions for This Repository

This is a Claude Code plugin marketplace. When adding plugins here, follow these conventions exactly.

## Public Repository Warning

This repository is public on GitHub. Never include:
- Real usernames, email addresses, or account names
- Internal hostnames, server names, or network paths
- Home directory paths (e.g., `/home/jdoe/`)
- API keys, tokens, passwords, or credentials
- Internal URLs (intranet, wiki, Jira, Confluence)
- Lab names or internal project codenames that are not already public

Use environment variables or clearly marked placeholders (`YOUR_USERNAME`, `your-server.example.com`) for any user-specific or environment-specific values.

## Naming Conventions

- **Plugin directories**: lowercase, hyphenated. e.g., `plugins/code-review/`
- **Skill directories**: lowercase, hyphenated. Match the skill name. e.g., `skills/code-review/`
- **Plugin names in `plugin.json` and `marketplace.json`**: must match the plugin directory name exactly
- **Skill names in `SKILL.md` frontmatter**: must match the skill directory name exactly
- Names should be descriptive and specific: `rails-test-runner` not `test-helper`, `api-doc-generator` not `docs`

## Adding a Plugin

### 1. Create the plugin directory

```
plugins/<plugin-name>/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── <skill-name>/
        └── SKILL.md
```

### 2. Write plugin.json

```json
{
  "name": "<plugin-name>",
  "description": "Clear one-line description of what this plugin does",
  "version": "1.0.0"
}
```

### 3. Write SKILL.md

```markdown
---
name: <skill-name>
description: Clear description of what this skill does and when to use it
---

Skill instructions here.
```

See https://code.claude.com/docs/en/skills for full SKILL.md authoring reference including frontmatter fields like `disable-model-invocation`, `allowed-tools`, `context`, and `agent`.

### 4. Register in marketplace.json

Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

```json
{
  "name": "<plugin-name>",
  "source": "<plugin-name>",
  "description": "Same description as plugin.json"
}
```

The `source` is relative to the `pluginRoot` defined in marketplace metadata (`./plugins`), so just the directory name is sufficient.

### 5. Open a PR

- Branch name: `add-<plugin-name>`
- PR title: `Add <plugin-name> plugin`
- PR body: describe what the plugin does and why it's useful
- One plugin per PR unless they are closely related

## Updating an Existing Plugin

- Bump the `version` in `plugin.json` (semver: patch for fixes, minor for new features, major for breaking changes)
- Update the `marketplace.json` entry if the description changed
- Branch name: `update-<plugin-name>`
- PR title: `Update <plugin-name> to <new-version>`

## Testing

Before opening a PR, verify the plugin loads correctly:

```bash
claude --plugin-dir ./plugins/<plugin-name>
```

Then test each skill with `/<plugin-name>:<skill-name>`.
