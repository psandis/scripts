# Cron & Automation Scripts

A curated collection of cron jobs, scheduled task patterns, and shell automation techniques.

---

## Table of Contents

- [Cron Basics](#cron-basics)
- [Cron Syntax](#cron-syntax)
- [Common Cron Examples](#common-cron-examples)
- [Crontab Management](#crontab-management)
- [macOS launchd](#macos-launchd)
- [Shell Script Patterns](#shell-script-patterns)
- [Logging & Notifications](#logging--notifications)
- [Tips & Best Practices](#tips--best-practices)

---

## Cron Basics

Cron is a time-based job scheduler. Jobs are defined in a `crontab` file and run automatically at specified intervals.

### Edit Your Crontab

```bash
crontab -e
```

### List Current Cron Jobs

```bash
crontab -l
```

### Remove All Cron Jobs

```bash
crontab -r
```

> ⚠️ This removes ALL cron jobs. There's no confirmation prompt.

### View Another User's Crontab (root)

```bash
sudo crontab -l -u username
```

---

## Cron Syntax

```
* * * * * /path/to/command
│ │ │ │ │
│ │ │ │ └── Day of week  (0-7, 0 and 7 = Sunday)
│ │ │ └──── Month        (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour         (0-23)
└────────── Minute       (0-59)
```

### Special Characters

| Symbol | Meaning | Example |
|--------|---------|---------|
| `*`    | Any value | `* * * * *` = every minute |
| `,`    | List | `1,15,30 * * * *` = at minute 1, 15, and 30 |
| `-`    | Range | `1-5 * * * *` = minutes 1 through 5 |
| `/`    | Step | `*/15 * * * *` = every 15 minutes |

### Special Strings

```bash
@reboot    # Run once at startup
@yearly    # Run once a year  (0 0 1 1 *)
@monthly   # Run once a month (0 0 1 * *)
@weekly    # Run once a week  (0 0 * * 0)
@daily     # Run once a day   (0 0 * * *)
@hourly    # Run once an hour (0 * * * *)
```

---

## Common Cron Examples

### Every Minute

```bash
* * * * * /path/to/script.sh
```

---

### Every 5 Minutes

```bash
*/5 * * * * /path/to/script.sh
```

---

### Every Hour at Minute 30

```bash
30 * * * * /path/to/script.sh
```

---

### Every Day at 2:00 AM

```bash
0 2 * * * /path/to/script.sh
```

---

### Every Monday at 9:00 AM

```bash
0 9 * * 1 /path/to/script.sh
```

---

### First Day of Every Month at Midnight

```bash
0 0 1 * * /path/to/script.sh
```

---

### Every Weekday (Mon–Fri) at 8:00 AM

```bash
0 8 * * 1-5 /path/to/script.sh
```

---

### At Reboot

```bash
@reboot /path/to/startup-script.sh
```

---

### Backup a Directory Daily

```bash
0 3 * * * tar -czf /backups/home-$(date +\%Y\%m\%d).tar.gz /home/user
```

> Note: Escape `%` as `\%` inside crontabs.

---

### Delete Files Older Than 30 Days

```bash
0 4 * * * find /tmp/logs -type f -mtime +30 -delete
```

---

### Restart a Service Every Night

```bash
0 0 * * * sudo systemctl restart nginx
```

---

### Sync Files to Remote Server

```bash
0 2 * * * rsync -az /var/www/ user@backup-server:/backups/www/
```

---

### Run a Node.js Script

```bash
*/10 * * * * /usr/local/bin/node /home/user/scripts/check.js >> /var/log/check.log 2>&1
```

---

## Crontab Management

### Redirect Output to Log File

```bash
* * * * * /path/to/script.sh >> /var/log/myjob.log 2>&1
```

- `>> /var/log/myjob.log` = append stdout to log
- `2>&1` = redirect stderr to stdout (capture errors too)

---

### Suppress All Output

```bash
* * * * * /path/to/script.sh > /dev/null 2>&1
```

---

### Set Environment Variables in Crontab

```bash
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=admin@example.com

0 * * * * /path/to/script.sh
```

---

### Use Full Paths

Always use absolute paths in cron because it runs in a minimal environment:

```bash
# Bad
0 * * * * node script.js

# Good
0 * * * * /usr/local/bin/node /home/user/script.js
```

---

## macOS launchd

macOS uses `launchd` instead of cron for system-level scheduling. User agents go in `~/Library/LaunchAgents/`.

### Example plist (run script every 5 minutes)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.user.myjob</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/user/scripts/myjob.sh</string>
  </array>

  <key>StartInterval</key>
  <integer>300</integer>

  <key>StandardOutPath</key>
  <string>/tmp/myjob.log</string>

  <key>StandardErrorPath</key>
  <string>/tmp/myjob.err</string>

  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

**Save to:** `~/Library/LaunchAgents/com.user.myjob.plist`

### Load / Unload the Agent

```bash
launchctl load ~/Library/LaunchAgents/com.user.myjob.plist
launchctl unload ~/Library/LaunchAgents/com.user.myjob.plist
```

### Check Status

```bash
launchctl list | grep com.user.myjob
```

---

## Shell Script Patterns

### Basic Script Template

```bash
#!/bin/bash
set -euo pipefail

LOG="/var/log/myjob.log"
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$TIMESTAMP] Starting job" >> "$LOG"

# Your commands here

echo "[$TIMESTAMP] Done" >> "$LOG"
```

- `set -e` = exit on error
- `set -u` = exit on undefined variable
- `set -o pipefail` = catch errors in pipes

---

### Lock File (Prevent Overlapping Runs)

```bash
#!/bin/bash
LOCKFILE="/tmp/myjob.lock"

if [ -f "$LOCKFILE" ]; then
  echo "Job already running. Exiting."
  exit 1
fi

touch "$LOCKFILE"
trap "rm -f $LOCKFILE" EXIT

# Your commands here
```

---

### Retry on Failure

```bash
#!/bin/bash
MAX_RETRIES=3
RETRY_DELAY=5

for i in $(seq 1 $MAX_RETRIES); do
  if your_command; then
    echo "Success on attempt $i"
    break
  else
    echo "Attempt $i failed. Retrying in ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
  fi
done
```

---

### Run Only on Weekdays

```bash
#!/bin/bash
DAY=$(date +%u)  # 1=Monday, 7=Sunday

if [ "$DAY" -ge 6 ]; then
  echo "Weekend - skipping"
  exit 0
fi

# Weekday-only commands here
```

---

### Timed Execution (Timeout)

```bash
# Kill command if it runs longer than 30 seconds
timeout 30 /path/to/slow-script.sh
```

---

## Logging & Notifications

### Rotating Log File

```bash
#!/bin/bash
LOG="/var/log/myjob.log"
MAX_SIZE=1048576  # 1 MB in bytes

if [ -f "$LOG" ] && [ $(stat -f%z "$LOG") -gt $MAX_SIZE ]; then
  mv "$LOG" "${LOG}.bak"
fi

echo "$(date): Running job" >> "$LOG"
```

---

### Send Email on Failure (requires mail)

```bash
#!/bin/bash
if ! /path/to/script.sh; then
  echo "Job failed at $(date)" | mail -s "Cron Job Failed" admin@example.com
fi
```

---

### macOS Notification on Completion

```bash
osascript -e 'display notification "Backup complete!" with title "Cron Job"'
```

---

### Log to System Log (macOS)

```bash
logger -t "myjob" "Job completed successfully"
```

**View it:**
```bash
log show --predicate 'senderImagePath contains "logger"' --last 1h
```

---

## Tips & Best Practices

- Always use **absolute paths** because cron doesn't load your shell profile
- **Test scripts manually** before scheduling them
- Redirect output to a log file so errors aren't silently lost
- Use a **lock file** to prevent overlapping runs for long-running jobs
- Use `@reboot` cautiously since errors at boot can be hard to debug
- Check `cron` is running: `sudo crontab -l` or `pgrep cron`
- On macOS, prefer **launchd** over cron for reliability
- Use [crontab.guru](https://crontab.guru) to verify your cron expressions
