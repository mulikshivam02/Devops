# 02. Arguments, Conditions, Loops and Functions

## 1. Positional arguments

Create:

```bash
script.sh
```

```bash
#!/usr/bin/env bash

echo "Script : $0"
echo "First  : $1"
echo "Second : $2"
echo "Count  : $#"
```

Run:

```bash
bash script.sh hello world
```

---

## 2. Validate required arguments

```bash
#!/usr/bin/env bash

if [[ $# -lt 1 ]]; then
    echo "Usage: $0 <name>" >&2
    exit 1
fi

NAME="$1"

echo "Hello, $NAME"
```

Run:

```bash
chmod +x script.sh
./script.sh Shivam
```

---

## 3. Optional argument with default

```bash
PORT="${1:-8080}"

echo "Port: $PORT"
```

Run:

```bash
./script.sh
./script.sh 9000
```

Change:
- `8080` -> default port
- `9000` -> value supplied at runtime

---

## 4. `if`

```bash
if [[ "$PORT" -eq 8080 ]]; then
    echo "Default port"
else
    echo "Custom port"
fi
```

String:

```bash
if [[ "$ENV" == "prod" ]]; then
    echo "Production"
fi
```

File:

```bash
if [[ -f "$FILE" ]]; then
    echo "File exists"
fi
```

---

## 5. `case`

Useful for commands/options:

```bash
case "${1:-}" in
    start)
        echo "Starting"
        ;;
    stop)
        echo "Stopping"
        ;;
    status)
        echo "Checking status"
        ;;
    *)
        echo "Usage: $0 {start|stop|status}"
        exit 1
        ;;
esac
```

---

## 6. AND / OR

```bash
command1 && command2
```

Run `command2` only if `command1` succeeds.

```bash
command1 || command2
```

Run `command2` if `command1` fails.

Example:

```bash
systemctl is-active --quiet nginx && echo "Running"
```

---

## 7. `for` loop

```bash
for service in nginx docker ssh; do
    echo "Checking $service"
done
```

Important: when expanding arrays, use:

```bash
for item in "${items[@]}"; do
    ...
done
```

---

## 8. Numeric loop

```bash
for ((i=1; i<=5; i++)); do
    echo "Number: $i"
done
```

---

## 9. `while`

```bash
COUNT=1

while [[ "$COUNT" -le 5 ]]; do
    echo "$COUNT"
    COUNT=$((COUNT + 1))
done
```

---

## 10. Read a file line by line

Preferred pattern:

```bash
while IFS= read -r line; do
    printf '%s\n' "$line"
done < input.txt
```

This preserves spaces and backslashes more safely than many naïve loops.

---

## 11. `break` and `continue`

```bash
for i in {1..10}; do
    if [[ "$i" -eq 5 ]]; then
        continue
    fi

    echo "$i"
done
```

Stop:

```bash
for i in {1..10}; do
    if [[ "$i" -eq 5 ]]; then
        break
    fi

    echo "$i"
done
```

---

## 12. Functions

```bash
log() {
    printf '[INFO] %s\n' "$*"
}

log "Application started"
```

Function with arguments:

```bash
greet() {
    local name="$1"
    echo "Hello, $name"
}

greet "Shivam"
```

Use `local` for function-local variables.

---

## 13. Function return status

```bash
check_file() {
    [[ -f "$1" ]]
}

if check_file "/etc/hosts"; then
    echo "File exists"
fi
```

Functions can return success/failure using command status or explicitly:

```bash
return 1
```

Avoid using `return` to pass large strings; command substitution or global/output mechanisms are usually clearer.

---

## 14. Command-line options with `getopts`

Basic example:

```bash
#!/usr/bin/env bash

PORT=8080
VERBOSE=false

while getopts ":p:v" opt; do
    case "$opt" in
        p)
            PORT="$OPTARG"
            ;;
        v)
            VERBOSE=true
            ;;
        *)
            echo "Usage: $0 [-p PORT] [-v]" >&2
            exit 1
            ;;
    esac
done

echo "PORT=$PORT"
echo "VERBOSE=$VERBOSE"
```

Run:

```bash
chmod +x script.sh

./script.sh
./script.sh -v
./script.sh -p 9000
./script.sh -p 9000 -v
```

### Fields to change

- `8080` = default port
- `PORT` = variable name used by the script
- `-p` = option for the port
- `-v` = verbose flag

---

## 15. Complete example: service checker

Create:

```bash
mkdir -p ~/shell-lab/service-check
cd ~/shell-lab/service-check

cat > check-services.sh <<'EOF'
#!/usr/bin/env bash

set -u

SERVICES=("ssh" "docker")

for service in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$service"; then
        echo "[OK]   $service"
    else
        echo "[FAIL] $service"
    fi
done
EOF

chmod +x check-services.sh
./check-services.sh
```

### Change these values

```bash
SERVICES=("ssh" "docker")
```

Replace/add service names:

```bash
SERVICES=("nginx" "docker" "postgresql")
```

Check available service names:

```bash
systemctl list-units --type=service
```

---

## 16. Complete example: argument validation

Create:

```bash
mkdir -p ~/shell-lab/validate
cd ~/shell-lab/validate

cat > validate-env.sh <<'EOF'
#!/usr/bin/env bash

set -u

if [[ $# -ne 2 ]]; then
    echo "Usage: $0 <environment> <port>" >&2
    exit 1
fi

ENV_NAME="$1"
PORT="$2"

case "$ENV_NAME" in
    dev|test|prod)
        ;;
    *)
        echo "Environment must be dev, test, or prod" >&2
        exit 1
        ;;
esac

if ! [[ "$PORT" =~ ^[0-9]+$ ]]; then
    echo "Port must be numeric" >&2
    exit 1
fi

if (( PORT < 1 || PORT > 65535 )); then
    echo "Port must be between 1 and 65535" >&2
    exit 1
fi

echo "Environment: $ENV_NAME"
echo "Port: $PORT"
```

Run:

```bash
chmod +x validate-env.sh

./validate-env.sh dev 8080
./validate-env.sh prod 443
./validate-env.sh abc 8080
```

---

## Common mistake

This is fragile:

```bash
for file in $(find . -type f); do
    ...
done
```

It breaks when filenames contain spaces/newlines.

For many tasks, prefer null-delimited processing:

```bash
find . -type f -print0 |
while IFS= read -r -d '' file; do
    printf '%s\n' "$file"
done
```

Or use `find ... -exec` where appropriate.
