# CLAUDE.md

Claude Code plugins repository. Plugins live under `plugins/`.

## Structure

```text
plugins/{plugin-name}/
├── .claude-plugin/
│   └── plugin.json               # Required: name, description, version, author
├── commands/
│   └── {command-name}.md         # Required: at least one command
└── README.md
```

Canonical example: `plugins/mco-tools/`

## Contributing Rules

- **Follow existing patterns.** Read `plugins/mco-tools/commands/migrate-tests.md` for command format.
- **Use kebab-case** for all plugin names and command files.
- **Register all plugins** in `.claude-plugin/marketplace.json`.
- **Bump `version`** in `plugin.json` when modifying plugin commands.
