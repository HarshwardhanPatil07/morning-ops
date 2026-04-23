# MCO Tools Plugin

Agent plugins for automating OpenShift Machine Config Operator (MCO) test workflows -- migration between repos and creation of new tests from Polarion specs.

## Commands

### /mco-tools:migrate-tests

Migrate MCO tests from `openshift-tests-private` to `machine-config-operator`.

**Features:**
- Two migration modes: whole file or suite extraction by keyword
- Test name transformation (Author format -> PolarionID format)
- Import rewriting (compat_otp -> exutil)
- Duplicate detection (skips already-migrated tests)
- Template/testdata and helper function migration
- Build verification and PR creation

**Usage:**
```bash
/mco-tools:migrate-tests
```

See [migrate-tests.md](./commands/migrate-tests.md) for full documentation.

### /mco-tools:automate-test

Create new MCO test cases from Polarion specifications, learning from previous code review feedback to produce review-ready code.

**Features:**
- Polarion-driven: translates test steps and expected results into executable Go test code
- Review-aware: analyzes past PR review comments to extract coding standards (cumulative, grows with each run)
- Convention-compliant: follows all MCO test patterns (naming, utilities, cleanup, error handling)
- Iterative: builds and fixes until the test compiles, then presents for user review
- Optional commit and PR creation

**Usage:**
```bash
/mco-tools:automate-test
```

See [automate-test.md](./commands/automate-test.md) for full documentation.

## Installation

Add this to your agent settings:

```json
{
  "extraKnownMarketplaces": {
    "morning-ops": {
      "source": {
        "source": "git",
        "url": "git@github.com:HarshwardhanPatil07/morning-ops.git"
      }
    }
  },
  "enabledPlugins": {
    "mco-tools@morning-ops": true
  }
}
```

## Requirements

- Go toolchain installed
- Git installed and configured
- `gh` CLI (for review learning in `automate-test`)
- Local clones of:
  - `openshift-tests-private`
  - `machine-config-operator`
  - (Optional) `origin` for compat_otp source

## Contributing

1. Add commands to `commands/<command-name>.md`
2. Bump `version` in `.claude-plugin/plugin.json`
3. Update this README
