---
description: Automate MCO test creation from Polarion specs, learning from previous code reviews
argument-hint: ""
---

## Name

mco-tools:automate-test

## Synopsis

```bash
/mco-tools:automate-test
```

## Description

The `mco-tools:automate-test` command automates the creation of new MCO (Machine Config Operator) test cases from Polarion test specifications. It combines Polarion spec analysis with lessons learned from previous code reviews in the `openshift-tests-private` repository to produce high-quality, review-ready test code.

**What it does:**

1. Collects repository paths and Polarion test spec from the user
2. Learns from previous code reviews by analyzing merged PR review comments in the MCO test directory
3. Reads existing test files to understand patterns, utilities, and conventions
4. Generates automated test code following all learned conventions
5. Builds and verifies the generated test compiles
6. Optionally commits and creates a PR

**Key Features:**

- **Polarion-driven** - Takes Polarion test case specifications (steps, expected results, preconditions) and translates them into executable Go test code
- **Review-aware** - Analyzes past code review comments on PRs in the MCO test directory to extract coding standards and avoid common review feedback
- **Convention-compliant** - Follows all established MCO test patterns (naming, utilities, error handling, resource cleanup)
- **Context-rich** - Reads the target test directory to understand existing helpers, utilities, and patterns before generating code
- **Iterative** - Builds and fixes until the test compiles, then presents the result for user review

## Implementation

**EXECUTE EACH PHASE IN ORDER:**

1. User Input Collection (repository paths, Polarion spec, target file)
2. Code Review Learning (analyze past PR reviews for coding standards)
3. Codebase Context (read existing tests, utilities, and patterns)
4. Test Generation (produce the test code)
5. Verification (build, format, user review)

### Phase 1: User Input Collection

Collect all necessary information from the user before starting.

**CRITICAL INSTRUCTIONS:**
- Ask each input explicitly using AskUserQuestion tool or direct prompts
- WAIT for user response before proceeding to the next input
- Validate paths exist before accepting them
- Check for a memory file named `automate_test_config.md` in the project memory directory -- if it exists, read the saved paths and offer them as defaults
- After paths are confirmed, run `git pull` on the repositories to ensure latest code

#### Input 1: Target Repository Path

If a saved path exists in memory, ask: "What is the path to your `openshift-tests-private` repository? (last used: `<saved-path>`, type `y` to reuse)"

Otherwise ask: "What is the path to your `openshift-tests-private` repository?"

**Validation:**
```bash
if [ ! -d "$TARGET_REPO/test/extended/mco" ]; then
    echo "ERROR: Cannot find test/extended/mco/ in the provided path"
    exit 1
fi
```

**Store in variable:** `<target-repo>`

**Set derived paths:**
```bash
TEST_DIR="$TARGET_REPO/test/extended/mco"
TESTDATA_DIR="$TARGET_REPO/test/extended/testdata/mco"
UTIL_DIR="$TARGET_REPO/test/extended/util/compat_otp"
```

**Save to memory immediately:** After validation passes, write (or update) the `automate_test_config.md` memory file with the target_repo path.

#### Input 1b: Sync Repository

```bash
cd "$TARGET_REPO" && git pull --ff-only 2>/dev/null || echo "WARNING: Could not pull repo (may have local changes)"
```

#### Input 2: Polarion Test Specification

Ask the user: "Please provide the Polarion test case specification. You can paste it directly or provide a URL/ID. Include:
- **Test Case ID** (Polarion ID number)
- **Title/Description**
- **Preconditions** (environment requirements, cluster state)
- **Test Steps** (numbered steps to execute)
- **Expected Results** (what each step should produce)
- **Tags/Labels** (priority, automation status, platform requirements)"

**Handling the response:**
- If the user provides a Polarion ID only (e.g., `OCP-12345` or just `12345`), ask them to paste the full spec
- If the user pastes a full spec, parse and extract all fields
- Store the parsed spec in structured variables

**Store in variables:**
- `<polarion-id>` = numeric ID (e.g., `12345`)
- `<test-title>` = test case title/description
- `<preconditions>` = preconditions text
- `<test-steps>` = ordered list of test steps
- `<expected-results>` = expected results per step
- `<tags>` = any tags/labels (platform, priority, etc.)

#### Input 3: Target File

Ask the user: "Which test file should this test be added to? Options:
- Enter an **existing filename** to add to it (e.g., `mco_configdrift.go`)
- Enter a **new filename** to create a new test file (e.g., `mco_newfeature.go`)
- Type `auto` to let me determine the best file based on the test topic"

**If `auto`:** Analyze the test title and preconditions to determine the best matching existing file by reading `g.Describe` block descriptions in all test files. If no good match, suggest a new filename.

**Validate:**
- If existing file: verify it exists in `$TEST_DIR`
- If new file: verify it ends in `.go` and doesn't conflict

**Store in variable:** `<target-file>` and `<is-new-file>` (true/false)

#### Input 4: Configuration Summary and Confirmation

```text
========================================
MCO Test Automation Configuration
========================================
Repository:      <target-repo>
Polarion ID:     <polarion-id>
Test Title:      <test-title>
Target File:     <target-file> (new/existing)
Platform Tags:   <tags if any>
========================================
Proceed? [Y/n]:
```

### Phase 2: Code Review Learning

Analyze previous code reviews on MCO test PRs to extract coding standards and common feedback patterns. This phase makes the generated test code review-ready by learning from real reviewer comments.

**CRITICAL: This phase is what differentiates this command from naive code generation. The AI must internalize review patterns to produce code that passes review on the first attempt.**

**How it works:** The `gh` CLI fetches review comments directly from GitHub's API. The repo does NOT need to be cloned locally for this phase — it only requires `gh` CLI installed and authenticated with read access to `openshift/openshift-tests-private`. The local repo path (from Phase 1) is used later for reading/writing test code, not for review learning.

#### Step 1: Load Existing Review Patterns from Memory

Check for the `review_patterns_mco.md` memory file:

- **If it exists**: Read it and note the `## Last Analyzed PR` number. This is the high-water mark — only PRs newer than this need to be fetched.
- **If it doesn't exist**: This is the first run. All recent PRs will be analyzed.

The review patterns memory is **cumulative** — it grows over time. Every run adds new patterns from new PRs. Old patterns are never deleted unless the user explicitly asks. This means the tool gets better with every use.

#### Step 2: Fetch New Merged PRs with Reviews

Fetch merged PRs that modified MCO test files, starting from after the last analyzed PR:

```bash
# If first run: get the last 30 merged PRs
gh pr list --repo openshift/openshift-tests-private \
  --state merged \
  --limit 30 \
  --search "mco test/extended/mco" \
  --json number,title,url,mergedAt \
  --jq '.[] | "\(.number) \(.title)"'

# If subsequent run: only get PRs newer than the last analyzed
# Filter results to only include PRs with number > <last-analyzed-pr-number>
```

If `gh` is not available:
1. Warn the user: "The `gh` CLI is not available. Review learning requires `gh` authenticated with read access to `openshift/openshift-tests-private`. Install it with: https://cli.github.com/"
2. If existing memory patterns exist, use those and continue to Phase 3
3. If no memory patterns exist, continue with built-in conventions only

If `gh` is available but authentication fails or repo access is denied:
1. Warn the user: "Cannot access `openshift/openshift-tests-private` via `gh`. Run `gh auth login` and ensure you have read access."
2. Fall back to existing memory patterns or built-in conventions

#### Step 3: Analyze Review Comments on New PRs

For each new PR not yet analyzed:

```bash
# Fetch inline review comments on MCO test files
gh api repos/openshift/openshift-tests-private/pulls/<PR_NUMBER>/comments \
  --jq '.[] | select(.path | test("test/extended/mco/")) | {path: .path, body: .body, line: .line}'
```

Also fetch PR-level review bodies:
```bash
gh api repos/openshift/openshift-tests-private/pulls/<PR_NUMBER>/reviews \
  --jq '.[] | select(.state == "CHANGES_REQUESTED" or .state == "COMMENTED") | .body'
```

**Skip PRs that have no review comments** — they don't contribute patterns.

#### Step 4: Extract and Categorize Review Patterns

From the review comments, extract actionable coding standards. Categorize each new pattern into:

1. **Naming conventions** - Variable names, function names, test descriptions
2. **Resource handling** - How to create, use, and clean up resources (MachineConfigs, MachineConfigPools, etc.)
3. **Error handling** - Logging patterns, assertion styles, error messages
4. **Code structure** - Step organization, helper function usage, defer patterns
5. **Common mistakes** - Things reviewers frequently ask to change

**Deduplication:** Before adding a new pattern, check if it duplicates or contradicts an existing pattern in memory:
- If it's a duplicate: skip it
- If it contradicts an existing pattern: the newer pattern wins (reviewer standards evolve). Update the existing pattern and note the PR that changed it
- If it's genuinely new: append it to the appropriate category

#### Step 5: Build Review Rules Document

Combine the accumulated review-extracted patterns (from memory + newly extracted) with the built-in conventions into a unified rules document for Phase 4.

**Built-in conventions (baseline rules):**

These are the foundational rules. Review-extracted patterns supplement and refine them:

1. **Read existing code first** - Before creating a new test case, read all files in the directory to understand what the existing code does
2. **Use GetCompactCompatiblePool** - Use `GetCompactCompatiblePool` function to get the MCP for testing, unless the test can only be executed in the master pool or instructed otherwise
3. **Use GetCurrentTestPolarionIDNumber** - Call `GetCurrentTestPolarionIDNumber()` to get the test ID. Call it only once per test
4. **Use Generic MC template** - Use the Generic MC template (default) when possible
5. **Base64 encode file configs** - Base64 encode file configuration in MachineConfigs unless said otherwise
6. **Use test number in identifiers** - Use test number in file paths, content, and resource names to ensure uniqueness
7. **Node selection** - Use `GetSortedNodesOrFail()[0]` for first node selection
8. **Step logging** - Log `"OK!\n"` after each step completes successfully
9. **Variables section** - Declare variables in the `vars` section if possible
10. **Error messages** - In error messages, log the full resource using `%s`. Prefer `logger.Infof("%s", node)` over `logger.Infof("%s", node.GetName())`
11. **AI-assisted comment** - Add `// AI-assisted` comment at the top of generated test code
12. **State recovery** - The initial state must always be recovered when a test case finishes. Use defer pattern: `defer machineConfiguration.SetSpec(machineConfiguration.GetSpecOrFail())`
13. **Prefer Resource struct methods** - Prioritize using methods on the Resource struct instead of `oc.AsAdmin.Run`
14. **Format code** - All files should be `gofmt`-ed with `-s`
15. **Concise comments** - Use short concise comments, one or two lines maximum
16. **RemoteFile for node verification** - Use `RemoteFile` with gomega checkers for verifying files inside nodes

#### Step 6: Save Updated Review Patterns to Memory

Update (or create) the `review_patterns_mco.md` memory file. This file is **append-only** for patterns — new patterns are added, old ones are never removed unless they are contradicted by newer reviews.

```markdown
---
name: MCO Test Review Patterns
description: Cumulative coding standards extracted from openshift-tests-private MCO PR reviews — grows with every run
type: project
---

## Last Analyzed PR: <highest PR number analyzed>
## Last Updated: <current date>
## Total PRs Analyzed: <cumulative count>

## Extracted Patterns

### Naming Conventions
- <pattern> (from PR #<number>)
- <pattern> (from PR #<number>)

### Resource Handling
- <pattern> (from PR #<number>)

### Error Handling
- <pattern> (from PR #<number>)

### Code Structure
- <pattern> (from PR #<number>)

### Common Mistakes to Avoid
- <pattern> (from PR #<number>)

## All Analyzed PRs
<cumulative list of all PR numbers ever analyzed, newest first>
```

**On every run:**
- Read the existing memory file
- Fetch only PRs newer than `Last Analyzed PR`
- Append new patterns (deduplicated) to the appropriate categories
- Update `Last Analyzed PR`, `Last Updated`, and `Total PRs Analyzed`
- Append new PR numbers to `All Analyzed PRs`

This ensures the knowledge base grows with every conversation and is available in every new context.

### Phase 3: Codebase Context

Read existing code to understand available utilities, patterns, and the target file structure.

#### Step 1: Read Target File (if existing)

If adding to an existing file, read the entire file to understand:
1. The `g.Describe` block structure and tags
2. The `g.JustBeforeEach` setup block
3. Variable declarations
4. Existing test cases and their patterns
5. Helper functions used
6. Import statements

#### Step 2: Read Utility Functions

Read the key utility packages to understand available helpers:

```bash
# Read the main MCO test utilities
ls "$TEST_DIR"/*.go | head -5
# Read compat_otp utilities
ls "$UTIL_DIR"/*.go 2>/dev/null | head -10
```

Focus on understanding:
1. **Resource structs** - MachineConfig, MachineConfigPool, MachineConfiguration, Node, etc.
2. **Common helper functions** - Pool waiting, node draining, MC creation, etc.
3. **Template utilities** - `generateTemplateAbsolutePath()`, template loading
4. **Assertion helpers** - Custom gomega matchers, RemoteFile checkers
5. **Setup/teardown patterns** - How existing tests handle state recovery

Read at minimum:
- The target test file (if existing)
- 2-3 similar test files in the same directory (based on test topic similarity)
- Key utility files that define Resource structs and common helpers

#### Step 3: Read Template Files

Check available testdata templates:

```bash
ls "$TESTDATA_DIR"/*.yaml 2>/dev/null | head -20
```

Read the Generic MC template and any templates relevant to the test being created.

#### Step 4: Identify Reusable Patterns

From the read code, identify:
1. How similar tests are structured (step-by-step flow)
2. Which helper functions handle the operations described in the Polarion steps
3. What assertions match the expected results
4. How cleanup/state recovery is implemented for similar resources
5. How the test ID number is used in resource naming

### Phase 4: Test Generation

Generate the test code based on Polarion spec, review patterns, and codebase context.

#### Step 1: Plan the Test Structure

Before writing code, create a mental outline:

1. **Test name**: `[PolarionID:<id>][OTP] <description>` -- or use the source repo format `Author:<username>-...-<id>-[tags] <description>` if targeting openshift-tests-private
2. **Variables needed**: MCP, node references, MachineConfig resources, file paths
3. **Setup**: What `g.JustBeforeEach` or inline setup is needed
4. **Steps**: Map each Polarion step to Go code using available helpers
5. **Assertions**: Map each expected result to gomega assertions
6. **Cleanup**: What defer statements are needed for state recovery

Determine the test name format based on the target repository:
- **If target is `openshift-tests-private`**: Use the source format:
  ```go
  g.It("Author:<username>-Longduration-NonPreRelease-High-<id>-[P1] <description>", func() {
  ```
  Ask the user for their author username if not already known (check memory first).

- **If target is `machine-config-operator`**: Use the migrated format:
  ```go
  g.It("[PolarionID:<id>][OTP] <description>", func() {
  ```

#### Step 2: Generate the Test Code

Write the test following all conventions. The generated code MUST:

```go
// AI-assisted
g.It("<test-name-per-format>", func() {
    // Get test ID - call only once
    testID := GetCurrentTestPolarionIDNumber()

    // Get a compatible MCP (unless master pool is required)
    mcp := GetCompactCompatiblePool(oc)

    // Variables section
    var (
        // Declare test-specific variables here
    )

    // Step 1: <Polarion step description>
    compat_otp.By("Step 1: <description>")
    // <implementation using Resource struct methods>
    logger.Infof("OK!\n")

    // Step 2: <Polarion step description>
    compat_otp.By("Step 2: <description>")
    // <implementation>
    logger.Infof("OK!\n")

    // ... more steps ...
})
```

**Generation rules:**

1. **Map Polarion steps to code**: Each Polarion test step becomes a `compat_otp.By()` / `exutil.By()` block
2. **Use existing helpers**: Prefer existing utility functions over writing raw kubectl/oc commands
3. **State recovery via defer**: Place cleanup defers immediately after resource creation
   ```go
   mc := NewMachineConfig(oc, "test-"+testID, mcp.GetName())
   defer mc.Delete()
   ```
4. **Use Generic MC template**: For MachineConfig creation, use the generic template unless the test requires specific ignition config
5. **Base64 encode file content**:
   ```go
   fileContent := base64.StdEncoding.EncodeToString([]byte("actual content"))
   ```
6. **Resource names include test ID**:
   ```go
   mcName := fmt.Sprintf("test-%s-mc", testID)
   filePath := fmt.Sprintf("/etc/test-%s-file", testID)
   ```
7. **Node verification with RemoteFile**:
   ```go
   remoteFile := NewRemoteFile(node, filePath)
   o.Expect(remoteFile).To(HaveContent("expected content"))
   ```
8. **Full resource in error messages**:
   ```go
   logger.Infof("MC status: %s", mc)
   // NOT: logger.Infof("MC name: %s", mc.GetName())
   ```
9. **Pool waiting after changes**:
   ```go
   mcp.WaitForComplete()
   ```

#### Step 3: Handle New File Creation

If `<is-new-file>` is true, generate the complete file:

```go
package mco

import (
    // standard library imports
    "encoding/base64"
    "fmt"

    // third-party imports
    g "github.com/onsi/ginkgo/v2"
    o "github.com/onsi/gomega"

    // project imports - adjust based on target repo
    compat_otp "github.com/openshift/origin/test/extended/util/compat_otp"
    logger "github.com/openshift/origin/test/extended/util/compat_otp/logext"
    // ... other imports as needed
)

var _ = g.Describe("[sig-mco] MCO <SuiteName>", func() {
    var (
        oc = compat_otp.NewCLI("mco")
    )

    g.JustBeforeEach(func() {
        // Standard setup
        preChecks(oc)
    })

    // AI-assisted
    g.It("<test-name>", func() {
        // ... generated test code ...
    })
})
```

#### Step 4: Handle Existing File Insertion

If adding to an existing file:
1. Read the file to find the correct insertion point (inside the `g.Describe` block, after the last `g.It` block)
2. Insert only the `g.It` block and any new helper functions
3. Add any new imports needed
4. Do not modify existing code

#### Step 5: Generate Testdata Templates (if needed)

If the test requires new YAML template files:
1. Create them in `$TESTDATA_DIR/`
2. Use the test ID in the template name for uniqueness
3. Follow the format of existing templates in the directory

### Phase 5: Verification

#### Step 1: Format the Code

```bash
cd "$TARGET_REPO"
goimports -w "$TEST_DIR/<target-file>"
# Fallback
gofmt -s -w "$TEST_DIR/<target-file>"
```

#### Step 2: Build the Test Binary

Build the test binary using `make build`:

```bash
cd "$TARGET_REPO"
make build
```

This produces the `./bin/extended-platform-tests` binary.

**On build failure:**
1. Read the error carefully
2. Common fixes:
   - Missing import: add it
   - Undefined function: check if a helper exists or needs to be written
   - Type mismatch: check the correct types from util packages
3. Fix and retry (up to 3 iterations)
4. If still failing after 3 attempts, show the error and ask the user for guidance

#### Step 3: Verify Test Listing (Dry Run)

After a successful build, verify the generated test appears in the binary by running a dry-run filtered by the Polarion ID:

```bash
cd "$TARGET_REPO"
./bin/extended-platform-tests run all --dry-run 2>&1 | grep "<polarion-id>" | head -5
```

**Expected output:** The test should appear with its full name, e.g.:
```text
"[sig-mco] MCO <SuiteName> Author:<username>-...-<polarion-id>-[P1] <description> [Serial]"
```

If the test does NOT appear:
1. Check `package mco` is correct
2. Check `var _ = g.Describe(...)` is used (init-time registration)
3. Check the file is in `test/extended/mco/`
4. Rebuild and retry

#### Step 4: Present Generated Code for Review

Show the user:
1. The complete generated test code (or diff if adding to existing file)
2. Any new template files created
3. Build status (pass/fail)
4. Dry-run listing output confirming the test is registered
5. A list of conventions applied
6. Any review patterns that influenced the code

Ask: "Please review the generated test. Would you like to:
1. **Accept** as-is
2. **Modify** - tell me what to change
3. **Regenerate** - start over with different approach"

#### Step 5: Iterate on Feedback

If the user requests modifications:
1. Apply the requested changes
2. Re-format and re-build (`make build`)
3. Re-verify with dry-run listing
4. Present the updated code
5. Repeat until accepted

**IMPORTANT:** If the user provides feedback that represents a new coding convention or correction, save it to the `review_patterns_mco.md` memory file so future runs benefit from it.

#### Step 6: Optional Test Execution Against a Cluster

After the user accepts the code, ask: "Would you like to run this test against a cluster? This requires a KUBECONFIG. [y/N]"

If yes:

```bash
cd "$TARGET_REPO"
export KUBECONFIG=~/Downloads/kubeconfig

# Run the specific test
./bin/extended-platform-tests run-test "<full-test-name>"
```

Where `<full-test-name>` is the complete test name from the dry-run listing (e.g., the full string shown by `grep "<polarion-id>"`).

**If the user has a custom KUBECONFIG path**, ask for it:
"What is the path to your kubeconfig? (default: `~/Downloads/kubeconfig`)"

Report the test result (pass/fail) and any output to the user.

#### Step 7: Optional Commit and PR

After user accepts the code, ask: "Would you like me to commit and create a PR? [Y/n]"

If yes:

**CRITICAL: Only stage test source files and testdata. NEVER stage `go.mod`, `go.sum`, or build artifacts (`bin/`, `_output/`).** The `make build` step may update `go.mod`/`go.sum` — these changes must NOT be committed. Restore them before committing:

```bash
cd "$TARGET_REPO"

# Restore go.mod and go.sum to their original state
git checkout -- go.mod go.sum 2>/dev/null

BRANCH="automate-test-<polarion-id>"
git checkout -b "$BRANCH"

# Only add test source files and testdata — nothing else
git add "$TEST_DIR/<target-file>"
git add "$TESTDATA_DIR/"*.yaml 2>/dev/null

git commit -s -m "$(cat <<'EOF'
Add automated test for OCP-<polarion-id>

Automated test case for Polarion ID <polarion-id>: <test-title>

AI-assisted test generation from Polarion specification.
EOF
)"
git push -u origin "$BRANCH"
gh pr create \
  --title "Add automated test for OCP-<polarion-id>" \
  --body "$(cat <<'EOF'
## Summary
- Automated test case for Polarion ID <polarion-id>
- <test-title>

## Test Details
- **Polarion ID:** <polarion-id>
- **File:** `test/extended/mco/<target-file>`
- **Steps:** <number of test steps>

## Generation Method
AI-assisted test generation from Polarion specification, following coding
conventions extracted from previous code reviews.

## Conventions Applied
<list key conventions that shaped the code>
EOF
)"
```

#### Step 6: Test Automation Summary

```text
========================================
MCO Test Automation Summary
========================================
Polarion ID:       <polarion-id>
Test Title:        <test-title>
Target File:       <target-file>
Test Steps:        <count>
Build:             PASSED/FAILED
Review Patterns:   <count> patterns applied
Branch:            <branch-name> (if committed)
PR:                <pr-url> (if created)

Conventions Applied:
  - GetCompactCompatiblePool for MCP selection
  - GetCurrentTestPolarionIDNumber called once
  - Generic MC template used
  - defer for state recovery
  - RemoteFile for node file verification
  - <additional review-learned patterns>

Next Steps:
  1. Review the generated test code
  2. Run against a cluster to verify behavior
  3. Address any PR review feedback
========================================
```

#### Step 7: Save Configuration to Memory

Update `automate_test_config.md` memory file:

```markdown
## Last-Used Paths
- target_repo: <target-repo>

## Last Generation
- date: <current date>
- polarion_id: <polarion-id>
- target_file: <target-file>
- build_status: PASSED/FAILED
```

### Troubleshooting

#### Build Error: undefined function

A utility function referenced in the generated code doesn't exist. Check:
1. The function name is correct (may be in a different package)
2. The import for the function's package is present
3. The function may need to be created as a helper

#### Build Error: unused import

An import was added but not needed. Run `goimports -w` to fix automatically.

#### Generated test doesn't match Polarion steps

Re-read the Polarion spec and verify each step has a corresponding `compat_otp.By()` / `exutil.By()` block. Ensure no steps were skipped or combined incorrectly.

#### Review patterns seem wrong or outdated

Individual patterns can be edited directly in the `review_patterns_mco.md` memory file. To do a full reset, delete the file and re-run — the command will re-fetch and re-analyze from scratch.

#### `gh` CLI not available or not authenticated

Review learning requires the `gh` CLI with read access to `openshift/openshift-tests-private`. The repo does NOT need to be cloned locally — `gh` fetches review data directly from GitHub's API. Install `gh` from https://cli.github.com/ and run `gh auth login`. Without `gh`, the command still works using built-in conventions and any previously saved patterns.

## Return Value

**On success:**
- Generated test code (shown to user for review)
- Build verification result
- List of conventions applied
- Optional: branch name and PR URL

**On failure:**
- Clear error message indicating which phase failed
- For build failures: error details and suggested fixes
- For missing context: what additional information is needed

## Examples

### Create a test from a Polarion spec

```text
User: /mco-tools:automate-test

AI: What is the path to your openshift-tests-private repository?
User: /home/user/repos/openshift-tests-private

AI: Please provide the Polarion test case specification.
User: OCP-12345
     Title: Verify MachineConfig with kernel arguments applied correctly
     Preconditions: Cluster with at least one worker node
     Steps:
       1. Create a MachineConfig with kernel argument "mitigations=auto"
       2. Wait for the MCP to finish updating
       3. Verify the kernel argument is present on the node
       4. Delete the MachineConfig
       5. Verify the kernel argument is removed
     Expected Results:
       1. MC created successfully
       2. MCP reports updated and complete
       3. /proc/cmdline contains "mitigations=auto"
       4. MC deleted
       5. /proc/cmdline no longer contains "mitigations=auto"

AI: Which test file should this test be added to?
User: mco.go

AI: [Analyzes code reviews, reads existing code, generates test]
    Here is the generated test:
    [shows code]
    Build: PASSED
    Would you like to accept, modify, or regenerate?
User: accept

AI: Would you like me to commit and create a PR?
User: y

AI: PR created: https://github.com/openshift/openshift-tests-private/pull/XXX
```

### Create a test in a new file

```text
AI: Which test file should this test be added to?
User: mco_newfeature.go

AI: Creating new file mco_newfeature.go with full test structure...
```

### Auto-select target file

```text
AI: Which test file should this test be added to?
User: auto

AI: Based on the test topic "kernel arguments", the best match is mco.go
    which already contains kernel-related tests. Use mco.go? [Y/n]
```

## Arguments

This command takes no arguments. All configuration is collected interactively during execution.

## Notes

- **Prerequisites**: Go toolchain, Git, `gh` CLI (for review learning), access to `openshift-tests-private` repository
- **Review learning**: The `gh` CLI fetches review comments directly from GitHub's API — the repo does NOT need to be cloned locally for this. It only requires `gh auth login` with read access to `openshift/openshift-tests-private`. Without `gh`, the command falls back to built-in conventions and any previously saved patterns
- **Convention priority**: Review-extracted patterns supplement the built-in conventions. If they conflict, review patterns take precedence (they represent the team's latest standards)
- **Cumulative memory**: Review patterns are stored permanently and grow with every run. Each run only fetches PRs newer than the last analyzed one, so it gets faster over time while the knowledge base keeps expanding. Delete `review_patterns_mco.md` to force a full re-analysis
- **AI-assisted comment**: All generated code includes an `// AI-assisted` comment for transparency
- **State recovery**: Every generated test includes proper cleanup via defer to restore initial state
- **Code preservation**: When adding to existing files, no existing code is modified

## See Also

- `mco-tools:migrate-tests` - Migrate existing tests between repositories
- OpenShift Tests Private: <https://github.com/openshift/openshift-tests-private>
- Machine Config Operator: <https://github.com/openshift/machine-config-operator>
