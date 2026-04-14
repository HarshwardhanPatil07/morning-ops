# MCO Tools Plugin

Tools for automating the migration of OpenShift Machine Config Operator (MCO) tests from `openshift-tests-private` to `machine-config-operator`.

## Commands

### /mco-tools:migrate-tests

Automate MCO test migration from openshift-tests-private to machine-config-operator.

**Features:**
- Two migration modes: whole file or suite extraction by keyword
- Accurate test name transformation (Author format → PolarionID format)
- Import rewriting (compat_otp → exutil)
- Duplicate detection (skips already-migrated tests)
- Template/testdata file migration
- Helper function migration
- Build verification
- PR creation automation

**Usage:**
```bash
/mco-tools:migrate-tests
```

See [migrate-tests.md](./commands/migrate-tests.md) for full documentation.

## Installation

Add this marketplace to `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "mco-tools@project-claude-kit": true
  }
}
```

## Requirements

- Go toolchain installed
- Git installed and configured
- Local clones of:
  - `openshift-tests-private`
  - `machine-config-operator`
  - (Optional) `origin` for compat_otp source

## Contributing

1. Add commands to `commands/<command-name>.md`
2. Bump `version` in `.claude-plugin/plugin.json`
3. Update this README
