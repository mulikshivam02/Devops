# 01. Shell Basics

## 1. What is a shell?

A shell is a command interpreter that lets you run programs and combine commands.

Common shells include:
- Bash
- `sh`
- Zsh
- Fish

Bash is the focus of this repository.

---

## 2. Check your shell

```bash
echo "$SHELL"
ps -p "$$" -o comm=
bash --version
```

### Important

`$SHELL` often shows the user's configured login shell. It is not always the exact shell currently executing a script.

---

## 3. Create a script

```bash
mkdir -p ~/shell-lab
cd ~/shell-lab

nano hello.sh
```

Put:

```bash
#!/usr/bin/env bash

echo "Hello World"
```

Save.

---

## 4. Run the script

### Method 1: explicitly invoke Bash

```bash
bash hello.sh
```

### Method 2: make executable

```bash
chmod +x hello.sh
./hello.sh
```

Check:

```bash
ls -l hello.sh
```

You should see executable bits such as `-rwxr-xr-x`.

---

## 5. Shebang

Recommended for Bash scripts:

```bash
#!/usr/bin/env bash
```

This tells the operating system which interpreter should execute the file.

Another common form:

```bash
#!/bin/bash
```

Use a shebang that matches the script's target shell.

---

## 6. Comments

```bash
#!/usr/bin/env bash

# This is a comment.
echo "Hello"
```

Comments are useful for:
- purpose
- assumptions
- values that need changing
- dangerous operations
- non-obvious logic

---

## 7. Variables

```bash
NAME="Shivam"
PORT=8080

echo "$NAME"
echo "$PORT"
```

There must be no spaces around `=`:

Correct:

```bash
NAME="Shivam"
```

Incorrect:

```bash
NAME = "Shivam"
```

---

## 8. Double quotes

Prefer quoting variables:

```bash
FILE="/tmp/my file.txt"

cat "$FILE"
```

Instead of:

```bash
cat $FILE
```

Quoting protects spaces and many special characters from accidental word splitting and pathname expansion.

---

## 9. Single vs double quotes

```bash
NAME="Shivam"

echo '$NAME'
echo "$NAME"
```

Output:

```text
$NAME
Shivam
```

Use single quotes when you want text to remain literal.

Use double quotes when variable/command expansion should happen.

---

## 10. Command substitution

Use:

```bash
CURRENT_DATE="$(date '+%Y-%m-%d')"
```

Then:

```bash
echo "$CURRENT_DATE"
```

Prefer `$(...)` over old-style backticks.

---

## 11. Arithmetic

```bash
A=10
B=20

SUM=$((A + B))

echo "$SUM"
```

Example:

```bash
COUNT=5
COUNT=$((COUNT + 1))
```

---

## 12. Environment variables

```bash
echo "$HOME"
echo "$USER"
echo "$PATH"
echo "$PWD"
```

Create an environment variable:

```bash
export APP_ENV="dev"
```

Check:

```bash
printenv APP_ENV
```

Environment variables are especially useful in CI/CD.

Example:

```bash
#!/usr/bin/env bash

echo "Environment: ${APP_ENV:-not-set}"
```

---

## 13. Default values

```bash
echo "${APP_PORT:-8080}"
```

Meaning:

- use `$APP_PORT` if set and non-empty
- otherwise use `8080`

Another important form:

```bash
: "${APP_PORT:?APP_PORT is required}"
```

This stops with a useful error if `APP_PORT` is unset or empty.

---

## 14. Arrays

```bash
SERVICES=("nginx" "docker" "ssh")

echo "${SERVICES[0]}"
echo "${SERVICES[1]}"
echo "${SERVICES[2]}"
```

Loop:

```bash
for service in "${SERVICES[@]}"; do
    echo "Checking: $service"
done
```

Always be careful with array quoting.

---

## 15. Read input

```bash
#!/usr/bin/env bash

read -r -p "Enter your name: " NAME

echo "Hello, $NAME"
```

Password:

```bash
read -r -s -p "Password: " PASSWORD
echo
```

---

## 16. Special parameters

```bash
echo "$0"
echo "$1"
echo "$2"
echo "$#"
echo "$@"
echo "$?"
echo "$$"
```

Meaning:

| Parameter | Meaning |
|---|---|
| `$0` | script name |
| `$1`, `$2` | positional arguments |
| `$#` | number of arguments |
| `$@` | all arguments |
| `$?` | previous command exit status |
| `$$` | current shell PID |

---

## 17. Source another file

Create:

```bash
config.sh
```

```bash
APP_NAME="myapp"
APP_PORT=8080
```

Then:

```bash
#!/usr/bin/env bash

source ./config.sh

echo "$APP_NAME"
echo "$APP_PORT"
```

Short form:

```bash
. ./config.sh
```

Only source trusted files.

---

## 18. Useful shell built-ins

```bash
help
help cd
help read
help printf
help test
help set
help export
help source
```

---

## 19. Command lookup

```bash
command -v bash
command -v docker
command -v kubectl
```

This is useful in scripts:

```bash
if ! command -v docker >/dev/null 2>&1; then
    echo "Docker is required" >&2
    exit 1
fi
```

---

## 20. File tests

```bash
FILE="/etc/hosts"

[[ -f "$FILE" ]]
[[ -d "/etc" ]]
[[ -r "$FILE" ]]
[[ -w "$FILE" ]]
[[ -x "/usr/bin/bash" ]]
```

Common tests:

| Test | Meaning |
|---|---|
| `-f` | regular file |
| `-d` | directory |
| `-e` | path exists |
| `-r` | readable |
| `-w` | writable |
| `-x` | executable |

---

## 21. Script debugging

Run:

```bash
bash -x script.sh
```

More syntax-focused checking:

```bash
bash -n script.sh
```

Do not treat `bash -n` as a complete correctness check; it mainly catches syntax errors.

---

## 22. Minimal reusable template

```bash
#!/usr/bin/env bash

set -euo pipefail

main() {
    echo "Starting..."
    # Add commands here
}

main "$@"
```

Learn what `set -euo pipefail` actually does before using it automatically in complex scripts.

---

## 23. Complete mini example

Create:

```bash
mkdir -p ~/shell-lab/basic
cd ~/shell-lab/basic

cat > system-info.sh <<'EOF'
#!/usr/bin/env bash

echo "Hostname : $(hostname)"
echo "User     : $USER"
echo "Date     : $(date '+%Y-%m-%d %H:%M:%S')"
echo "Uptime   : $(uptime -p)"
echo "Kernel   : $(uname -r)"
echo "Shell    : ${SHELL:-unknown}"
EOF

chmod +x system-info.sh
./system-info.sh
```

### Values to change

This one does not require modification.

It is a good starting template for scripts that gather environment information.
