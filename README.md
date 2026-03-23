# Whitehead Claude Code Plugins

Plugin marketplace for [Claude Code](https://code.claude.com/) at Whitehead Institute.

## Install

```
/plugin marketplace add whitehead/claude-plugins
```

Then browse available plugins:

```
/plugin search @whitehead
```

## Available Plugins

*Coming soon.*

## Adding a Plugin

Create a directory under `plugins/` with the standard plugin structure:

```
plugins/my-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── my-skill/
        └── SKILL.md
```

Then add an entry to `.claude-plugin/marketplace.json`.
