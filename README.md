# Project Claude Kit

A collection of Claude Code plugins for OpenShift development and testing workflows.

## Overview

This repository serves as a marketplace for Claude Code plugins that automate common development tasks.

## Plugins

### MCO Tools

Automates migration of OpenShift MCO tests from `openshift-tests-private` to `machine-config-operator`.

**Commands:**
- `/migrate-tests` - Migrate tests between repositories with full transformation

## Installation

### Option 1: Add to Claude Settings (Recommended)

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

Then restart Claude Code to load the plugins.

### Option 2: Clone Locally

```bash
# Clone to Claude plugins directory
git clone git@github.com:HarshwardhanPatil07/project-claude-kit.git \
  ~/.claude/plugins/marketplaces/project-claude-kit

# Or clone anywhere and symlink
git clone git@github.com:HarshwardhanPatil07/project-claude-kit.git ~/repos/project-claude-kit
ln -s ~/repos/project-claude-kit ~/.claude/plugins/marketplaces/project-claude-kit
```

Then add to `settings.json`:

```json
{
  "enabledPlugins": {
    "mco-tools@project-claude-kit": true
  }
}
```

## Usage

Once installed, you can use the MCO tools in any Claude Code conversation:

```
/migrate-tests
```

This will guide you through migrating MCO tests with:
- Interactive configuration collection
- Automatic code transformation
- Duplicate detection
- Build verification
- PR creation

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
│       │   └── migrate-tests.md # Command with full workflow
│       └── README.md
├── .gitignore
└── README.md                    # This file
```

## Adding New Plugins

To add a new plugin to this marketplace:

1. Create plugin directory structure:
   ```bash
   mkdir -p plugins/<plugin-name>/.claude-plugin
   mkdir -p plugins/<plugin-name>/commands
   ```

2. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`:
   ```json
   {
     "name": "<plugin-name>",
     "description": "Description of your plugin",
     "version": "0.1.0",
     "author": {
       "name": "Your Name"
     }
   }
   ```

3. Add commands with proper frontmatter:
   ```markdown
   ---
   description: One-line description
   argument-hint: ""
   ---
   ```

4. Register in `.claude-plugin/marketplace.json`:
   ```json
   {
     "plugins": [
       {
         "name": "<plugin-name>",
         "source": "./plugins/<plugin-name>",
         "description": "Description",
         "version": "0.1.0"
       }
     ]
   }
   ```

5. Enable in `~/.claude/settings.json`:
   ```json
   {
     "enabledPlugins": {
       "<plugin-name>@project-claude-kit": true
     }
   }
   ```

6. Commit and push changes

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Add your plugin
4. Submit a pull request

## Resources

- [Claude Code Documentation](https://docs.anthropic.com/claude-code)
- [Example Plugins](https://github.com/anthropics/claude-plugins-official)
- [OpenShift MCO Repo](https://github.com/openshift/machine-config-operator)
- [OpenShift Tests Private](https://github.com/openshift/openshift-tests-private)

## License

Apache 2.0 - See LICENSE file for details

## Author

HarshwardhanPatil07

## Support

For issues or questions:
- Open an issue on GitHub
- Check existing plugins in [claude-plugins-official](https://github.com/anthropics/claude-plugins-official) for examples
- Review the MCO tools documentation in `plugins/mco-tools/`
