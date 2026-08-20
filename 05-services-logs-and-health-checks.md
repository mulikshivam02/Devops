# 05. Services, Logs and Health Checks

## 1. systemd

Many modern Linux distributions use systemd.

Basic service status:

```bash
systemctl status nginx
```

Start:

```bash
sudo systemctl start nginx
```

Stop:

```bash
sudo systemctl stop nginx
```

Restart:

```bash
sudo systemctl restart nginx
```

Reload:

```bash
sudo systemctl reload nginx
```

Enable at boot:

```bash
sudo systemctl enable nginx
```

Enable and start:

```bash
sudo systemctl enable --now nginx
```

Disable:

```bash
sudo systemctl disable nginx
```

---

## 2. Script-friendly service checks

Prefer:

```bash
systemctl is-active --quiet nginx
```

Example:

```bash
if systemctl is-active --quiet nginx; then
    echo "nginx is running"
else
    echo "nginx is not running"
fi
```

---

## 3. Check whether a unit exists

```bash
systemctl cat nginx.service
```

or:

```bash
systemctl list-unit-files | grep nginx
```

---

## 4. Logs with journalctl

Recent logs:

```bash
journalctl -u nginx
```

Recent entries:

```bash
journalctl -u nginx -n 50
```

Follow:

```bash
journalctl -u nginx -f
```

Since boot:

```bash
journalctl -u nginx -b
```

Since a time:

```bash
journalctl -u nginx --since "1 hour ago"
```

---

## 5. Search service logs

```bash
journalctl -u nginx --since today | grep -i error
```

Count errors:

```bash
journalctl -u nginx --since today | grep -ic error
```

---

## 6. Complete service health check

Create:

```text
examples/05-service-health-check/health-check.sh
```

```bash
#!/usr/bin/env bash

set -u

SERVICE="${1:-nginx}"
URL="${2:-http://127.0.0.1/}"

echo "Checking service: $SERVICE"
echo "Checking URL:     $URL"

if systemctl is-active --quiet "$SERVICE"; then
    echo "[OK] service is active"
else
    echo "[FAIL] service is not active"
    exit 1
fi

if curl -fsS --max-time 10 "$URL" >/dev/null; then
    echo "[OK] HTTP health check passed"
else
    echo "[FAIL] HTTP health check failed"
    exit 1
fi

echo "[OK] health check passed"
```

Run:

```bash
chmod +x health-check.sh
./health-check.sh nginx http://127.0.0.1/
```

### Change

```bash
SERVICE="${1:-nginx}"
URL="${2:-http://127.0.0.1/}"
```

You can run:

```bash
./health-check.sh docker
```

for a service-only check if the script is modified accordingly, or provide a real HTTP service for the URL test.

For an app on port 8080:

```bash
./health-check.sh myapp http://127.0.0.1:8080/health
```

---

## 7. Restart on failure pattern

A basic automation pattern:

```bash
#!/usr/bin/env bash

SERVICE="nginx"

if ! systemctl is-active --quiet "$SERVICE"; then
    echo "$(date): $SERVICE is down; attempting restart" >&2
    sudo systemctl restart "$SERVICE"
fi
```

For production, consider whether systemd's native `Restart=` behavior is more appropriate than a periodic shell script.

---

## 8. HTTP health check

```bash
curl -fsS --max-time 10 http://127.0.0.1:8080/health
```

Useful flags:
- `-f`: fail on HTTP errors
- `-s`: silent
- `-S`: show errors when silent
- `--max-time`: timeout

Check only status:

```bash
curl -fsS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/health
```

---

## 9. Complete log-monitor example

Create:

```text
examples/06-log-monitor/log-monitor.sh
```

```bash
#!/usr/bin/env bash

set -u

LOG_FILE="${1:-app.log}"
PATTERN="${2:-ERROR}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "Log file not found: $LOG_FILE" >&2
    exit 1
fi

echo "Monitoring: $LOG_FILE"
echo "Pattern:    $PATTERN"
echo "Press Ctrl+C to stop."

tail -Fn0 "$LOG_FILE" |
while IFS= read -r line; do
    if grep -qi -- "$PATTERN" <<< "$line"; then
        printf '[MATCH] %s\n' "$line"
    fi
done
```

Run:

```bash
chmod +x log-monitor.sh
./log-monitor.sh /var/log/myapp.log ERROR
```

### Change

- first argument = log path
- second argument = search pattern

---

## 10. Test the log monitor locally

Create sample log:

```bash
printf '%s\n' "INFO started" "INFO ready" > app.log
```

Start monitor:

```bash
./log-monitor.sh app.log ERROR
```

In another terminal:

```bash
echo "ERROR database unavailable" >> app.log
echo "INFO retrying" >> app.log
echo "ERROR timeout" >> app.log
```

The monitor should print matching lines.
