# 04. Processes, Resources and System Monitoring

This is one of the most useful Shell Scripting areas for DevOps.

---

## 1. System overview

```bash
hostname
uname -a
uname -r
uptime
```

---

## 2. CPU information

```bash
nproc
lscpu
```

Current load:

```bash
uptime
```

Load average can also be read from:

```bash
cat /proc/loadavg
```

---

## 3. Memory

```bash
free -h
```

Useful fields include:
- total
- used
- free
- available

Do not interpret `used` in isolation; Linux uses memory for cache and buffers.

---

## 4. Disk space

```bash
df -h
```

Check root:

```bash
df -h /
```

Script-friendly filesystem list:

```bash
df -P
```

---

## 5. Directory usage

```bash
du -sh /var/log
```

Top entries:

```bash
du -xh /var/log 2>/dev/null | sort -h | tail -20
```

---

## 6. Processes

List processes:

```bash
ps aux
```

Custom columns:

```bash
ps -eo pid,ppid,comm,%cpu,%mem --sort=-%cpu
```

Top CPU processes:

```bash
ps -eo pid,comm,%cpu --sort=-%cpu | head
```

Top memory:

```bash
ps -eo pid,comm,%mem --sort=-%mem | head
```

---

## 7. Find a process

```bash
pgrep -af nginx
```

Alternative:

```bash
ps aux | grep '[n]ginx'
```

`pgrep` is usually cleaner for process lookup.

---

## 8. Kill a process

First identify the PID carefully:

```bash
pgrep -af <PROCESS_NAME>
```

Graceful termination:

```bash
kill <PID>
```

Force only when necessary:

```bash
kill -9 <PID>
```

Do not put `kill -9` into automation unless you understand the failure mode.

---

## 9. Open files

```bash
lsof
```

Example:

```bash
sudo lsof -i :8080
```

---

## 10. Network

Interfaces:

```bash
ip addr
```

Routes:

```bash
ip route
```

Ports:

```bash
ss -tulpn
```

Check a port:

```bash
ss -ltn 'sport = :8080'
```

---

## 11. Connectivity checks

```bash
ping -c 4 8.8.8.8
```

DNS:

```bash
getent hosts example.com
```

HTTP:

```bash
curl -I https://example.com
```

---

## 12. Resource monitoring script

Create:

```text
examples/04-resource-monitor/resource-monitor.sh
```

```bash
#!/usr/bin/env bash

set -u

INTERVAL="${1:-5}"

if ! [[ "$INTERVAL" =~ ^[0-9]+$ ]]; then
    echo "Usage: $0 [interval_seconds]" >&2
    exit 1
fi

while true; do
    clear

    echo "===== SYSTEM ====="
    date
    echo "Hostname: $(hostname)"
    echo "Uptime:   $(uptime -p)"

    echo
    echo "===== MEMORY ====="
    free -h

    echo
    echo "===== DISK ====="
    df -h /

    echo
    echo "===== LOAD ====="
    cat /proc/loadavg

    echo
    echo "===== TOP CPU ====="
    ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head -n 11

    echo
    echo "Refreshing every ${INTERVAL}s. Press Ctrl+C to stop."

    sleep "$INTERVAL"
done
```

Run:

```bash
chmod +x resource-monitor.sh
./resource-monitor.sh
```

Custom interval:

```bash
./resource-monitor.sh 2
```

### Change

```bash
INTERVAL="${1:-5}"
```

- `5` = default refresh interval in seconds
- `2` = refresh every 2 seconds

---

## 13. Threshold alert script

Create:

```text
cpu-memory-check.sh
```

```bash
#!/usr/bin/env bash

set -u

CPU_THRESHOLD="${CPU_THRESHOLD:-80}"
MEMORY_THRESHOLD="${MEMORY_THRESHOLD:-80}"
ROOT_DISK_THRESHOLD="${ROOT_DISK_THRESHOLD:-80}"

cpu_usage="$(LC_ALL=C top -bn1 | awk '/Cpu\(s\)/ {print 100 - $8}')"
memory_usage="$(free | awk '/Mem:/ {printf "%.0f", $3/$2 * 100}')"
disk_usage="$(df -P / | awk 'NR==2 {gsub("%",""); print $5}')"

echo "CPU:    ${cpu_usage}%"
echo "Memory: ${memory_usage}%"
echo "Disk:   ${disk_usage}%"

awk "BEGIN {exit !($cpu_usage >= $CPU_THRESHOLD)}" &&
    echo "WARNING: CPU above ${CPU_THRESHOLD}%"

if (( memory_usage >= MEMORY_THRESHOLD )); then
    echo "WARNING: Memory above ${MEMORY_THRESHOLD}%"
fi

if (( disk_usage >= ROOT_DISK_THRESHOLD )); then
    echo "WARNING: / above ${ROOT_DISK_THRESHOLD}%"
fi
```

Run:

```bash
chmod +x cpu-memory-check.sh
./cpu-memory-check.sh
```

Override thresholds:

```bash
CPU_THRESHOLD=70 MEMORY_THRESHOLD=75 ROOT_DISK_THRESHOLD=85 ./cpu-memory-check.sh
```

### Important

Resource parsing commands can differ across operating systems and tool versions. Test on the target environment before using in production monitoring.

---

## 14. Log resource values

```bash
./cpu-memory-check.sh | tee -a resource-history.log
```

Timestamped:

```bash
{
    echo "===== $(date '+%Y-%m-%d %H:%M:%S') ====="
    ./cpu-memory-check.sh
} | tee -a resource-history.log
```

---

## 15. Use `watch` before writing a loop

For quick interactive monitoring:

```bash
watch -n 2 'free -h; echo; df -h /; echo; uptime'
```

Use a shell script when you need:
- thresholds
- logging
- alerts
- automation
- custom logic

---

## 16. Process monitoring example

```bash
#!/usr/bin/env bash

PROCESS="${1:-nginx}"

if pgrep -x "$PROCESS" >/dev/null; then
    echo "RUNNING: $PROCESS"
else
    echo "NOT RUNNING: $PROCESS"
    exit 1
fi
```

Run:

```bash
./check-process.sh nginx
```

Change:
- `nginx` -> process/service name

---

## 17. Avoid brittle parsing

Do not parse human-oriented output unless necessary.

Prefer machine-readable options when available:
- `ps -eo ...`
- `df -P`
- `systemctl is-active`
- `pgrep`
- `/proc/*` where appropriate

Always test against the target Linux distribution.
