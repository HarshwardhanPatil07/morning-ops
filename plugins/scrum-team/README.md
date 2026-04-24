# scrum-team

Multi-agent scrum team plugin for Claude Code. Give **Harshya** (Team Lead) a software development task and the team gets it done.

## The Team

| Agent | Role |
|-------|------|
| **Harshya** (Team Lead) | Receives your task, picks the right specialist, coordinates the workflow, mediates cross-agent communication |
| Architect | Analyzes the codebase, designs solutions, identifies files to change and risks |
| Developer | Implements code changes — writes, edits, builds, and verifies |
| QE | Writes and runs tests, validates changes, reports failures with root cause analysis |
| Staff Engineer | Reviews all changes for correctness, style, architecture — approves or requests changes |

## How It Works

1. You describe a task (feature, bug fix, refactor, test gap, review)
2. Harshya classifies it and picks the starting agent — not always Architect
3. Agents work sequentially, each building on the previous agent's output
4. If an agent has a question for another agent, Harshya mediates the exchange
5. Staff Engineer reviews at the end — if changes are needed, the loop continues
6. Harshya reports the final summary

```
Bug fix:     Developer → QE → Staff Engineer
New feature: Architect → Developer → QE → Staff Engineer
Test gap:    QE → (Developer if needed) → Staff Engineer
Review:      Staff Engineer → (Developer if needed)
```

## Usage

```bash
/scrum-team:run "add a --dry-run flag to daily-meeting-reminder.sh"
/scrum-team:run "fix the off-by-one error in the date calculation"
/scrum-team:run "write tests for the idempotency check logic"
/scrum-team:run "review the security audit section for prompt injection gaps"
```

## Installation

Add to your Claude Code settings:

```json
{
  "projects": {
    "/path/to/your/project": {
      "extraKnownMarketplaces": {
        "morning-ops": {
          "source": {
            "source": "git",
            "url": "git@github.com:HarshwardhanPatil07/morning-ops.git"
          }
        }
      },
      "enabledPlugins": {
        "scrum-team@morning-ops": true
      }
    }
  }
}
```

## Prerequisites

- Claude Code with plugin support
- The Agent tool must be available (standard in Claude Code)
