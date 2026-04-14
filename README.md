# Project Claude Kit

A collection of Claude Code plugins for OpenShift development and testing workflows.

## Plugins

### MCO Tools

Automates migration of OpenShift MCO tests from `openshift-tests-private` to `machine-config-operator`.

**Commands:**
- `/mco-tools:migrate-tests` — Migrate tests between repositories with full transformation

## Installation

Add this repository as a marketplace in your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "project-claude-kit": {
      "source": {
        "source": "git",
        "url": "git@github.com:HarshwardhanPatil07/project-claude-kit.git"
      }
    }
  },
  "enabledPlugins": {
    "mco-tools@project-claude-kit": true
  }
}
```

## Repository Structure

```
project-claude-kit/
├── .claude-plugin/
│   └── marketplace.json         # Marketplace configuration
├── plugins/
│   └── mco-tools/               # MCO test migration tools
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── commands/
│       │   └── migrate-tests.md
│       └── README.md
├── CLAUDE.md                    # Plugin development guide
├── .gitignore
└── README.md
```

## Adding New Plugins

1. Create plugin directory:
   ```bash
   mkdir -p plugins/<plugin-name>/.claude-plugin
   mkdir -p plugins/<plugin-name>/commands
   ```

2. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`:
   ```json
   {
     "name": "<plugin-name>",
     "description": "Description of your plugin",
     "version": "0.0.1",
     "author": {
       "name": "Your Name"
     }
   }
   ```

3. Add commands as `commands/<command-name>.md` with frontmatter:
   ```markdown
   ---
   description: One-line description
   argument-hint: ""
   ---
   ```

4. Register in `.claude-plugin/marketplace.json`

5. Enable in `~/.claude/settings.json`:
   ```json
   {
     "enabledPlugins": {
       "<plugin-name>@project-claude-kit": true
     }
   }
   ```

## Author

HarshwardhanPatil07

## License

Apache 2.0
