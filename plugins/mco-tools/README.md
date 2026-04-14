# MCO Tools Plugin

Tools for automating the migration of OpenShift Machine Config Operator (MCO) tests from `openshift-tests-private` to `machine-config-operator`.

## Overview

This plugin automates the complex process of migrating MCO test cases between repositories, handling all necessary transformations including package renaming, import rewriting, test name reformatting, template file copying, and utility function migration.

## Commands

### /migrate-tests

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
/migrate-tests
```

The command will interactively guide you through:
1. Collecting repository paths
2. Selecting migration target (file or file:keyword)
3. Analyzing duplicates and conflicts
4. Confirming migration plan
5. Executing transformations
6. Building and verifying tests
7. Creating PRs

See [migrate-tests.md](./commands/migrate-tests.md) for full documentation.

## Installation

This plugin is part of the `project-claude-kit` marketplace. See the main repository README for installation instructions.

Once installed, enable it in `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "mco-tools@project-claude-kit": true
  }
}
```

## Examples

### Migrate a whole test file

```bash
/migrate-tests
# Source: /home/user/repos/openshift-tests-private
# Dest: /home/user/repos/machine-config-operator
# Target: mco_configdrift.go
```

### Extract tests by keyword

```bash
/migrate-tests
# Source: /home/user/repos/openshift-tests-private
# Dest: /home/user/repos/machine-config-operator
# Target: mco.go:kernel
# Creates: mco_kernel.go with all kernel-related tests
```

## Requirements

- Go toolchain installed
- Git installed and configured
- Local clones of:
  - `openshift-tests-private`
  - `machine-config-operator`
  - (Optional) `origin` for compat_otp source

## Migration Process

The migration handles:

1. **Package transformation**: `package mco` → `package extended`
2. **Import rewriting**: `compat_otp` → `exutil`
3. **Test name format**: `Author:USER-...-ID-[Tags] Desc` → `[PolarionID:ID][OTP] Desc`
4. **Suite tags**: Adds `[Suite:openshift/machine-config-operator/longduration][Serial][Disruptive]`
5. **Helper functions**: Migrates dependencies and utilities
6. **Template files**: Copies testdata from source to destination
7. **Build verification**: Ensures migrated tests compile and list correctly

## Contributing

To add new tools or improve existing ones:

1. Add commands to `commands/<command-name>.md`
2. Update plugin version in `.claude-plugin/plugin.json`
3. Update this README

## License

Apache 2.0 - See main repository LICENSE file
