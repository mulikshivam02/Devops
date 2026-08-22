# 06. Automation, Cron and Scheduling

## 1. Why automate with shell scripts?

Common repetitive tasks:
- backups
- cleanup
- log rotation helpers
- health checks
- resource reports
- deployment wrappers
- environment checks
- temporary file cleanup
- scheduled reports

---

## 2. Check cron service

On systemd-based systems:

```bash
systemctl status cron
```

Some distributions use:

```bash
systemctl status crond
```

---

## 3. User crontab

Edit:

```bash
crontab -e
```

List:

```bash
crontab -l
```

---

## 4. Cron format

```text
minute hour day-of-month month day-of-week command
```

Example:

```cron
*/5 * * * * /home/user/scripts/check.sh
```

Runs every 5 minutes.

---

## 5. Important: use absolute paths

Bad:

```cron
*/5 * * * * ./check.sh
```

Better:

```cron
*/5 * * * * /home/USER/scripts/check.sh
```

Cron has a different environment from your interactive shell.

Your script should use:
- absolute paths
- explicit environment values
- explicit interpreter
- reliable logging

---

## 6. Redirect cron output

```cron
*/5 * * * * /home/USER/scripts/check.sh >> /home/USER/logs/check.log 2>&1
```

Change:
- `USER`
- script path
- log path

---

## 7. Example scheduled health check

Create:

```text
~/scripts/health-check.sh
```

```bash
#!/usr/bin/env bash

set -u

URL="http://127.0.0.1:8080/health"
LOG_FILE="$HOME/health-check.log"

if curl -fsS --max-time 10 "$URL" >/dev/null; then
    printf '[%s] OK\n' "$(date '+%Y-%m-%d %H:%M:%S')" >> "$LOG_FILE"
else
    printf '[%s] FAIL %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$URL" >> "$LOG_FILE"
fi
```

Create:

```bash
mkdir -p "$HOME/scripts"
nano "$HOME/scripts/health-check.sh"
chmod +x "$HOME/scripts/health-check.sh"
```

Test:

```bash
"$HOME/scripts/health-check.sh"
cat "$HOME/health-check.log"
```

Add to crontab:

```cron
*/5 * * * * /home/USER/scripts/health-check.sh
```

Replace `/home/USER` with your actual path:

```bash
echo "$HOME"
```

---

## 8. Cron with environment variables

Do not assume interactive variables are present.

Better script:

```bash
#!/usr/bin/env bash

set -u

APP_URL="${APP_URL:-http://127.0.0.1:8080/health}"

curl -fsS "$APP_URL"
```

Cron line:

```cron
*/5 * * * * APP_URL=http://127.0.0.1:9000/health /home/USER/scripts/health-check.sh
```

---

## 9. Prevent overlapping runs

For scripts that may run longer than the schedule, use a locking strategy.

Simple pattern:

```bash
LOCK_FILE="/tmp/my-script.lock"

if (set -o noclobber; > "$LOCK_FILE") 2>/dev/null; then
    trap 'rm -f "$LOCK_FILE"' EXIT
else
    echo "Already running"
    exit 1
fi
```

For more robust locking on Linux, `flock` is often preferable:

```bash
flock -n /tmp/my-script.lock -c '/home/USER/scripts/my-script.sh'
```

---

## 10. Better: systemd timers

For systemd systems, a systemd service + timer can be preferable to cron when you need:
- dependency management
- richer logs
- service state
- calendar/monotonic timers
- resource controls

Basic service:

```ini
[Unit]
Description=My health check

[Service]
Type=oneshot
ExecStart=/home/USER/scripts/health-check.sh
```

Timer:

```ini
[Unit]
Description=Run health check periodically

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Unit=my-health-check.service

[Install]
WantedBy=timers.target
```

Install:

```bash
mkdir -p ~/.config/systemd/user
cp my-health-check.service ~/.config/systemd/user/
cp my-health-check.timer ~/.config/systemd/user/

systemctl --user daemon-reload
systemctl --user enable --now my-health-check.timer
```

Check:

```bash
systemctl --user list-timers
```

Inspect:

```bash
systemctl --user status my-health-check.timer
journalctl --user -u my-health-check.service
```

---

## 11. Which to use?

Use cron when:
- the job is simple
- basic scheduling is enough

Use systemd timer when:
- the host already uses systemd
- you want integrated service logging/state
- you want service dependencies or richer control

Use CI/CD schedulers when:
- automation belongs to a repository pipeline rather than the server itself

---

## 12. Automation checklist

Before scheduling:

```text
[ ] Absolute script path
[ ] Script executable
[ ] Correct shebang
[ ] Environment variables explicitly defined
[ ] Logs configured
[ ] Failures return non-zero
[ ] Permissions checked
[ ] Overlap considered
[ ] Tested manually
[ ] Tested under the scheduler's environment
```
