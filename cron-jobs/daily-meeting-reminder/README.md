# Daily Meeting Reminder

Reads your Google Calendar every weekday morning, finds previous meeting notes in Drive, and emails you a prepared brief with action items -- all before your day starts.

## What You Get

A styled HTML email for each meeting with:

- Title, time, clickable Google Meet link, and attendees
- Summary of the previous occurrence (from Google Docs)
- Carry-forward action items and open follow-ups
- Links to meeting notes and Gemini auto-generated notes
- A desktop notification with a quick summary of the day

## How It Works

1. **systemd timer** fires at 8 AM Mon-Fri (`Persistent=true` -- catches up if the machine was off)
2. **Shell script** orchestrates the run with concurrency locking and idempotency checks
3. **OpenCode** (`opencode run`) uses `gws` (Google Workspace CLI) to fetch events, search Drive, read docs, and send the email
4. If the AI agent fails, a **fallback email** is sent so you always get a reminder
5. On Fridays, it fetches Monday's meetings automatically

## Reliability and Security

- **No duplicate emails** -- sent-marker files + `flock` concurrency lock
- **Systemd sandboxing** -- `ProtectSystem=strict`, `NoNewPrivileges=true`, scoped write paths
- **Read-only credentials** -- OAuth files bind-mounted read-only
- **Post-run audit** -- logs scanned for unauthorized recipients and suspicious commands
- **Hardcoded recipient** -- email address set in the script, not in the AI prompt
- **Timezone-aware** -- uses local offset to avoid date boundary issues

## Prerequisites

- OpenCode -- AI agent that orchestrates the workflow
- gws -- Google Workspace CLI (Calendar, Drive, Gmail, Docs)

## Setup

> **Don't want to set up manually?** Just tell your AI agent to do it -- paste this README and it will handle the setup for you.

### 1. Authenticate gws

```bash
gws auth login
```

Needs scopes: Calendar (read), Drive (search), Docs (read), Gmail (send).

### 2. Install OpenCode

Follow [opencode.ai](https://opencode.ai). Verify with `opencode --version`.

### 3. Configure the script

Edit the config block at the top of `daily-meeting-reminder.sh`:

```bash
EMAIL_TO="your-email@example.com"
OPENCODE_BIN="/usr/local/bin/opencode"
OPENCODE_MODEL="provider/model-name"
OPENCODE_TIMEOUT=360
```

Then place it and make it executable:

```bash
cp daily-meeting-reminder.sh ~/.local/bin/
chmod +x ~/.local/bin/daily-meeting-reminder.sh
```

### 4. Create systemd units

<details>
<summary>Timer: ~/.config/systemd/user/daily-meeting-reminder.timer</summary>

```ini
[Unit]
Description=Daily Meeting Reminder Timer - 8:00 AM weekdays

[Timer]
OnCalendar=Mon..Fri *-*-* 08:00:00
Persistent=true
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
```

</details>

<details>
<summary>Service: ~/.config/systemd/user/daily-meeting-reminder.service</summary>

```ini
[Unit]
Description=Daily Meeting Reminder - fetch calendar, send prep email
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=%h/.local/bin/daily-meeting-reminder.sh
TimeoutStartSec=480
Restart=no

# Sandboxing
ProtectSystem=strict
ProtectHome=tmpfs
NoNewPrivileges=true
ProtectKernelModules=true
ProtectKernelTunables=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

# Write access (minimal)
ReadWritePaths=%h/.local/share/daily-meeting-reminder
ReadWritePaths=%h/.local/share/opencode
ReadWritePaths=%h/.config/opencode
ReadWritePaths=/tmp
ReadWritePaths=%h/.config/gws/token_cache.json

# Read-only credential access
BindReadOnlyPaths=%h/.config/gws/client_secret.json
BindReadOnlyPaths=%h/.config/gws/credentials.enc
BindReadOnlyPaths=%h/.config/gws/.encryption_key

# Paths the script and opencode need
BindPaths=%h/.local:%h/.local
BindPaths=%h/.config:%h/.config
BindPaths=%h/.agents:%h/.agents

PrivateTmp=false

[Install]
WantedBy=default.target
```

Adjust `BindReadOnlyPaths` and `ReadWritePaths` to match your credential locations.

</details>

### 5. Enable and test

```bash
systemctl --user daemon-reload
systemctl --user enable --now daily-meeting-reminder.timer
systemctl --user list-timers  # verify it's active
```

Test manually:

```bash
~/.local/bin/daily-meeting-reminder.sh today
```

## Usage

```bash
./daily-meeting-reminder.sh              # next working day (default)
./daily-meeting-reminder.sh today        # today's meetings
./daily-meeting-reminder.sh tomorrow     # tomorrow's meetings
./daily-meeting-reminder.sh "2026-04-21" # specific date
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| gws auth expired | Run `gws auth login` to re-authenticate |
| OpenCode times out | Increase `OPENCODE_TIMEOUT` in the script |
| No desktop notification | Ensure `DISPLAY` and `DBUS_SESSION_BUS_ADDRESS` are set; install `libnotify` |
| Duplicate emails | Delete the marker at `~/.local/share/daily-meeting-reminder/sent-markers/YYYY-MM-DD.sent` |
| Timer didn't fire | Re-enable: `systemctl --user enable --now daily-meeting-reminder.timer` |
| Logs | `~/.local/share/daily-meeting-reminder/logs/` (auto-cleaned after 30 days) |
