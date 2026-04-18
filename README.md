# Project Agentic Harshyaa

Your mornings, automated.

An AI-powered toolkit that prepares your workday before you even open your laptop. It reads your Google Calendar, digs up previous meeting notes, extracts carry-forward action items, and sends you a fully prepared meeting brief by email -- every weekday at 8 AM. It also includes AI-driven plugins for automating OpenShift MCO test workflows.

---

## The Problem

Engineers lose 15-20 minutes every morning doing the same thing: open the calendar, scan the day's meetings, try to remember what happened last time, hunt through Google Docs for notes, mentally piece together what needs follow-up. Multiply that across a five-day week, a team of ten, and a quarter -- and you're looking at hundreds of hours spent on something a machine can do better.

The real cost isn't even the time. It's walking into a meeting unprepared because you didn't have five minutes to look up the last one.

---

## What You Get

A styled HTML email in your inbox every weekday morning, before your day starts. One email, all meetings covered.

For each meeting on your calendar, the email includes:

- Meeting title, time, and Google Meet link (clickable)
- List of attendees
- A summary of what was discussed in the previous occurrence
- Carry-forward action items and open follow-ups
- Direct links to the relevant Google Docs (meeting notes, Gemini auto-generated notes)

You also get a desktop notification with a quick summary of the day's meetings.

Here's what a meeting card in the email looks like:

```
+------------------------------------------------------------------+
|  1. MCO Standup                                                  |
|------------------------------------------------------------------|
|  Time:         14:30 - 14:45 IST                                 |
|  Google Meet:  https://meet.google.com/abc-defg-hij              |
|  Attendees:    alice@example.com, bob@example.com                |
|------------------------------------------------------------------|
|  Previous Meeting Summary:                                       |
|  Discussed migration blockers for the control-plane tests.       |
|  Agreed to split the PR into smaller reviewable chunks.          |
|------------------------------------------------------------------|
|  Carry-forward Action Items:                                     |
|  - Alice to update the helper function signatures by Thursday    |
|  - Bob to file the upstream bug for the flaky e2e test           |
|------------------------------------------------------------------|
|  Previous Notes Doc  |  Gemini Notes                             |
+------------------------------------------------------------------+
```

---

## How It Works

```
+-----------------+     +-------------------+     +------------------+
|  systemd timer  |     |   Shell script    |     |   OpenCode       |
|  (8 AM Mon-Fri) +---->+  (orchestrator)   +---->+   AI agent       |
|  Persistent=true|     |  concurrency lock |     |  (opencode run)  |
+-----------------+     |  idempotency check|     +--------+---------+
                        |  fallback logic   |              |
                        |  security audit   |              | invokes
                        +-------------------+              v
                                                  +------------------+
                                                  |   gws CLI        |
                                                  |  (Google APIs)   |
                                                  +--------+---------+
                                                           |
                                          +----------------+----------------+
                                          |                |                |
                                          v                v                v
                                   +-----------+   +-----------+   +-----------+
                                   |  Calendar  |   |   Drive   |   |   Gmail   |
                                   |  (read)    |   |  (search) |   |  (send)   |
                                   +-----------+   +-----------+   +-----------+
```

**systemd timer** fires at 8:00 AM every weekday. If the machine was asleep or off, it catches up as soon as it wakes (`Persistent=true`).

**Shell script** is the orchestrator. It handles concurrency locking (only one instance at a time), idempotency checks (won't send duplicate emails for the same date), environment setup, and error recovery. If the AI agent fails or times out, the script sends a basic fallback email so you're never left without a reminder.

**OpenCode** (`opencode run`) is the AI coding agent. It receives a structured prompt describing exactly what to do -- fetch calendar events, search Drive for notes, read document content, compose the email, send it. The agent uses `gws` (Google Workspace CLI) to interact with Google APIs.

**gws** handles all Google API calls: reading Calendar events, searching Drive for documents, reading Docs content, and sending Gmail. It manages OAuth authentication locally.

On Fridays, the script automatically fetches Monday's meetings so you can prepare over the weekend if you want. You can also run it manually for any date.

---

## Why You Can Trust It to Run Unattended

### Reliability

- **Persistent timer.** If your laptop was off or asleep at 8 AM, the automation runs as soon as the machine wakes up. You never miss a day.
- **Idempotency markers.** A sent-marker file is written after each successful email. If the service restarts or crashes mid-run, it won't send duplicate emails for the same date.
- **Fallback email.** If the AI agent times out or fails, the script sends a basic "check your calendar manually" email. You always get something.
- **Concurrency lock.** Uses `flock` to ensure only one instance runs at a time, even if systemd retries or you trigger a manual run simultaneously.

### Security

- **Systemd sandboxing.** The service runs with `ProtectSystem=strict`, `NoNewPrivileges=true`, and carefully scoped `ReadWritePaths`. The agent can only write to its own data directory, not to your home directory at large.
- **Read-only credentials.** OAuth credential files are bind-mounted as read-only. The agent can authenticate but cannot modify or exfiltrate your credentials.
- **Post-execution audit.** After every run, the script scans its own logs for signs of prompt injection -- unauthorized email recipients, suspicious commands (`curl`, `wget`, `rm -rf`, access to `~/.ssh`). If anything looks wrong, it fires a critical desktop notification.
- **Hardcoded recipient.** The email recipient is set in the shell script, not in the AI prompt. The security audit verifies no emails were sent to any other address.

### Usability

- **Timezone-aware.** Uses your local timezone offset instead of UTC, so early-morning meetings in IST (or any timezone ahead of UTC) aren't accidentally assigned to the previous day.
- **Flexible invocation.** Run it for today, tomorrow, a specific date, or let it auto-detect the next working day:
  ```bash
  ./daily-meeting-reminder.sh              # next working day (default)
  ./daily-meeting-reminder.sh today        # today's meetings
  ./daily-meeting-reminder.sh tomorrow     # tomorrow
  ./daily-meeting-reminder.sh "2026-04-21" # specific date
  ```
- **Desktop notification.** A system notification pops up with the meeting summary, so you see it even before opening your email.
- **Logs with run IDs.** Every execution gets a unique run ID. Logs are timestamped and stored at `~/.local/share/daily-meeting-reminder/logs/`. Old logs and markers are cleaned up automatically after 30 days.

---

## Get It Running

This takes about 15 minutes on a Linux machine with systemd.

### Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| [OpenCode](https://opencode.ai) | AI coding agent that orchestrates the workflow | See [opencode.ai](https://opencode.ai) |
| [gws](https://github.com/nicholasgasior/gws) | Google Workspace CLI for Calendar, Drive, Gmail, Docs | See gws documentation |
| systemd | Service and timer management | Pre-installed on most Linux distributions |
| notify-send | Desktop notifications (optional) | `sudo dnf install libnotify` or `sudo apt install libnotify-bin` |

### Step 1: Install and authenticate gws

After installing `gws`, authenticate with the Google Workspace scopes the automation needs:

```bash
gws auth login
```

The automation requires these API scopes:
- Google Calendar (read events)
- Google Drive (search files)
- Google Docs (read document content)
- Gmail (send email)

Verify it works:

```bash
gws calendar events list --params '{"calendarId":"primary","maxResults":1}' --format json
```

### Step 2: Install OpenCode

Follow the instructions at [opencode.ai](https://opencode.ai). Verify:

```bash
opencode --version
```

### Step 3: Create the automation script

Create the script at `~/.local/bin/daily-meeting-reminder.sh`. The script has five sections:

**Configuration block** -- set these variables to match your environment:

```bash
EMAIL_TO="your-email@example.com"
OPENCODE_BIN="/usr/local/bin/opencode"
OPENCODE_MODEL="provider/model-name"
OPENCODE_TIMEOUT=360  # seconds
```

**Date resolution** -- determines the target date. On weekdays it fetches tomorrow's meetings. On Fridays it fetches Monday's. Accepts command-line overrides (`today`, `tomorrow`, `YYYY-MM-DD`).

**Prompt** -- a structured prompt sent to `opencode run` that tells the AI agent exactly what to do:
1. Fetch calendar events for the target date using `gws calendar events list`
2. Search Drive for previous meeting notes using `gws drive files list`
3. Read the last 150 lines of any found notes doc using `gws docs documents get`
4. Compose and send a styled HTML email using `gws gmail +send`
5. Print a plain-text summary for the desktop notification

**Execution** -- runs `opencode run` with the prompt, captures output, handles timeouts and failures with fallback email logic.

**Post-execution** -- security audit (scans for unauthorized recipients and suspicious commands), email verification (checks for Gmail `threadId` in logs), summary extraction, desktop notification, and log cleanup.

Make it executable:

```bash
chmod +x ~/.local/bin/daily-meeting-reminder.sh
```

### Step 4: Create the systemd timer

Create `~/.config/systemd/user/daily-meeting-reminder.timer`:

<details>
<summary>Timer unit file</summary>

```ini
[Unit]
Description=Daily Meeting Reminder Timer - 8:00 AM weekdays

[Timer]
# Run at 8:00 AM Monday-Friday
OnCalendar=Mon..Fri *-*-* 08:00:00
# If the laptop was off at 8 AM, run as soon as it wakes up
Persistent=true
# Small random delay to avoid exact-second spikes
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
```

</details>

### Step 5: Create the systemd service

Create `~/.config/systemd/user/daily-meeting-reminder.service`:

<details>
<summary>Service unit file</summary>

```ini
[Unit]
Description=Daily Meeting Reminder - fetch calendar, send prep email
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=%h/.local/bin/daily-meeting-reminder.sh

# Timeout must exceed script timeout + fallback timeout + buffer
TimeoutStartSec=480

# Do NOT auto-restart. The script has its own fallback logic.
# Restarting risks duplicate emails.
Restart=no

# --- Sandboxing ---
ProtectSystem=strict
ProtectHome=tmpfs
NoNewPrivileges=true
ProtectKernelModules=true
ProtectKernelTunables=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# --- Write access (minimal) ---
ReadWritePaths=%h/.local/share/daily-meeting-reminder
ReadWritePaths=%h/.local/share/opencode
ReadWritePaths=%h/.config/opencode
ReadWritePaths=/tmp

# gws token refresh needs write access
ReadWritePaths=%h/.config/gws/token_cache.json

# --- Read-only credential access ---
BindReadOnlyPaths=%h/.config/gws/client_secret.json
BindReadOnlyPaths=%h/.config/gws/credentials.enc
BindReadOnlyPaths=%h/.config/gws/.encryption_key

# --- Paths the script and opencode need ---
BindPaths=%h/.local:%h/.local
BindPaths=%h/.config:%h/.config
BindPaths=%h/.agents:%h/.agents

PrivateTmp=false

[Install]
WantedBy=default.target
```

</details>

Adjust the `BindReadOnlyPaths` and `ReadWritePaths` to match where your `gws` credentials and `opencode` data live.

### Step 6: Enable and start

```bash
systemctl --user daemon-reload
systemctl --user enable --now daily-meeting-reminder.timer
```

Verify the timer is active:

```bash
systemctl --user list-timers
```

### Step 7: Test it

Run manually for today's meetings:

```bash
~/.local/bin/daily-meeting-reminder.sh today
```

Or trigger the service directly:

```bash
systemctl --user start daily-meeting-reminder.service
journalctl --user -u daily-meeting-reminder.service -f
```

Check the logs:

```bash
cat ~/.local/share/daily-meeting-reminder/logs/$(date +%Y-%m-%d).log
```

---

## Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `EMAIL_TO` | -- | Recipient email address. The only address emails will ever be sent to. |
| `OPENCODE_BIN` | `/usr/local/bin/opencode` | Path to the OpenCode binary. |
| `OPENCODE_MODEL` | -- | AI model to use (e.g., `provider/model-name`). |
| `OPENCODE_TIMEOUT` | `360` | Maximum seconds to wait for the AI agent to finish. |
| `OPENCODE_WORKSPACE` | `~/.local/share/daily-meeting-reminder/workspace` | Isolated directory for OpenCode session data. |
| `BASE_DIR` | `~/.local/share/daily-meeting-reminder` | Root directory for logs, markers, and output. |
| `LOCK_FILE` | `/tmp/daily-meeting-reminder-$(id -u).lock` | Concurrency lock file path. |

---

## MCO Tools Plugin

This repository also includes Claude Code plugins for automating OpenShift Machine Config Operator (MCO) test workflows.

### Available Commands

| Command | Description |
|---------|-------------|
| `/mco-tools:migrate-tests` | Migrate MCO tests from `openshift-tests-private` to `machine-config-operator` with full transformation (name rewriting, import rewriting, duplicate detection, helper migration, PR creation) |
| `/mco-tools:automate-test` | Create new MCO test cases from Polarion specifications, learning from previous code review feedback to produce review-ready code |

### Installation

Add this repository as a plugin source in your agent's settings:

```json
{
  "extraKnownMarketplaces": {
    "project-agentic-harshyaa": {
      "source": {
        "source": "git",
        "url": "git@github.com:HarshwardhanPatil07/project-agentic-harshyaa.git"
      }
    }
  },
  "enabledPlugins": {
    "mco-tools@project-agentic-harshyaa": true
  }
}
```

See [plugins/mco-tools/README.md](plugins/mco-tools/README.md) for detailed documentation on each command.

---

## Troubleshooting

**gws authentication expired**
Run `gws auth login` to re-authenticate. The automation will fail with API 401 errors if the token has expired and can't be refreshed.

**OpenCode times out**
Increase `OPENCODE_TIMEOUT` in the script. The default is 360 seconds (6 minutes). Complex calendars with many meetings and large notes docs may need more time. The fallback email will still be sent on timeout.

**No desktop notification**
The notification requires a running display session. Check that `DISPLAY` and `DBUS_SESSION_BUS_ADDRESS` environment variables are set. Install `libnotify` if `notify-send` is not found.

**Duplicate emails**
The script writes a sent-marker file at `~/.local/share/daily-meeting-reminder/sent-markers/YYYY-MM-DD.sent` after each successful send. If you need to re-send for a date, delete the corresponding marker file.

**Timer didn't fire**
Check that the timer is enabled: `systemctl --user list-timers`. If the timer shows no upcoming activation, re-enable it: `systemctl --user enable --now daily-meeting-reminder.timer`.

**Logs location**
All logs are at `~/.local/share/daily-meeting-reminder/logs/`. Each day gets its own log file. The full AI agent output from the last run is saved to `~/.local/share/daily-meeting-reminder/last-opencode-output.txt`.

---

## Repository Structure

```
project-agentic-harshyaa/
├── .claude-plugin/
│   └── marketplace.json             # Plugin marketplace configuration
├── plugins/
│   └── mco-tools/                   # MCO test automation plugins
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── commands/
│       │   ├── migrate-tests.md     # Test migration command
│       │   └── automate-test.md     # Polarion-driven test creation
│       └── README.md
├── CLAUDE.md                        # Plugin development guide
├── .gitignore
└── README.md                        # This file
```

---

## Author

[HarshwardhanPatil07](https://github.com/HarshwardhanPatil07)

## License

Apache 2.0
