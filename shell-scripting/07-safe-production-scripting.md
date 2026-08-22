# 07. Safe and Production-Style Bash Scripting

## 1. `set -euo pipefail`

Common starting point:

```bash
set -euo pipefail
```

It enables:
- `-e`: exit on certain unhandled command failures
- `-u`: treat unset variables as errors
- `pipefail`: pipeline status can reflect earlier failures

Do not assume this combination makes every Bash script automatically safe. Bash `errexit` has documented exceptions and can behave differently inside conditionals, lists, functions, and subshells.

---

## 2. Example

```bash
#!/usr/bin/env bash

set -euo pipefail

echo "Start"
mkdir -p /tmp/example
echo "End"
```

---

## 3. `pipefail`

Without `pipefail`, a pipeline normally uses the last command's exit status.

Example:

```bash
false | true
echo "$?"
```

Normally the final `true` succeeds.

With:

```bash
set -o pipefail
false | true
echo "$?"
```

the pipeline can report failure because the earlier command failed.

This matters heavily in automation.

---

## 4. `trap`

Cleanup:

```bash
TMP_DIR="$(mktemp -d)"

cleanup() {
    rm -rf "$TMP_DIR"
}

trap cleanup EXIT
```

Now cleanup runs when the shell exits normally or because of many exit paths.

---

## 5. Temporary files

Prefer:

```bash
TMP_FILE="$(mktemp)"
trap 'rm -f "$TMP_FILE"' EXIT
```

Avoid hard-coded predictable temporary filenames such as:

```bash
/tmp/output.txt
```

when multiple users/processes could collide or a file could be replaced unexpectedly.

---

## 6. Error logging

```bash
log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}
```

Use stderr for errors:

```bash
echo "Something failed" >&2
```

---

## 7. Command existence check

```bash
require_command() {
    if ! command -v "$1" >/dev/null 2>&1; then
        echo "Required command not found: $1" >&2
        exit 1
    fi
}

require_command curl
require_command jq
```

---

## 8. Root check

Only use when truly required:

```bash
if [[ "$EUID" -ne 0 ]]; then
    echo "Run this script as root." >&2
    exit 1
fi
```

Better is often to require `sudo` only for the specific command that needs privilege.

---

## 9. Dry-run pattern

Useful for deployment/cleanup scripts:

```bash
DRY_RUN=false

run() {
    if "$DRY_RUN"; then
        printf '+'
        printf ' %q' "$@"
        printf '\n'
    else
        "$@"
    fi
}
```

Example:

```bash
run rm -rf -- "$TARGET"
```

Set:

```bash
DRY_RUN=true
```

to print actions instead of executing them.

---

## 10. Quoting

Prefer:

```bash
rm -- "$FILE"
```

instead of:

```bash
rm $FILE
```

Use `--` where supported to reduce ambiguity when a value begins with `-`.

---

## 11. Validate before destructive actions

Bad:

```bash
rm -rf "$TARGET"
```

Better:

```bash
if [[ -z "${TARGET:-}" ]]; then
    echo "TARGET is empty" >&2
    exit 1
fi

if [[ "$TARGET" == "/" ]]; then
    echo "Refusing to delete root" >&2
    exit 1
fi
```

Never add a "safety check" without understanding all allowed inputs.

---

## 12. ShellCheck

Install on Debian/Ubuntu:

```bash
sudo apt update
sudo apt install shellcheck
```

Run:

```bash
shellcheck script.sh
```

For a whole repository:

```bash
find . -type f -name '*.sh' -print0 | xargs -0 shellcheck
```

Or use your preferred repository CI integration.

ShellCheck detects many common quoting, syntax, semantic, robustness, and portability issues.

---

## 13. Syntax check

```bash
bash -n script.sh
```

Then:

```bash
shellcheck script.sh
```

Then test the script.

---

## 14. Debug execution

```bash
bash -x script.sh
```

For a specific section:

```bash
set -x
# commands to debug
set +x
```

---

## 15. Safe script template

```bash
#!/usr/bin/env bash

set -euo pipefail

readonly APP_NAME="my-app"
readonly DEFAULT_PORT="8080"

log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

die() {
    printf '[ERROR] %s\n' "$*" >&2
    exit 1
}

require_command() {
    command -v "$1" >/dev/null 2>&1 ||
        die "Required command not found: $1"
}

main() {
    require_command curl
    log "Starting ${APP_NAME}"
    # Work here.
}

main "$@"
```

---

## 16. Important Bash pitfalls

### Word splitting

Avoid:

```bash
for file in $FILES; do
```

Prefer arrays.

### Globbing

Unquoted patterns may expand unexpectedly.

### `eval`

Avoid `eval` unless you have a strong reason and understand the security implications.

### Parsing `ls`

Do not use:

```bash
for file in $(ls)
```

Prefer shell globs or `find`.

### Parsing command output

Prefer stable/machine-friendly options when available.

### Untrusted input

Do not blindly execute user-controlled strings.

---

## 17. `IFS`

For controlled parsing, you may temporarily use:

```bash
IFS=',' read -r USERNAME ROLE <<< "$LINE"
```

Example:

```bash
LINE="alice,admin"

IFS=',' read -r USERNAME ROLE <<< "$LINE"

echo "$USERNAME"
echo "$ROLE"
```

---

## 18. Arrays for command arguments

Bad:

```bash
OPTIONS="-a -l"
ls $OPTIONS
```

Better:

```bash
OPTIONS=(-a -l)
ls "${OPTIONS[@]}"
```

This is especially important when arguments can contain spaces.

---

## 19. Complete production-style example

Create:

```text
examples/07-backup-automation/backup.sh
```

```bash
#!/usr/bin/env bash

set -euo pipefail

SOURCE_DIR="${SOURCE_DIR:-$HOME/data}"
BACKUP_DIR="${BACKUP_DIR:-$HOME/backups}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"

log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

die() {
    printf '[ERROR] %s\n' "$*" >&2
    exit 1
}

require_command() {
    command -v "$1" >/dev/null 2>&1 ||
        die "Missing command: $1"
}

main() {
    require_command tar
    require_command find

    [[ -d "$SOURCE_DIR" ]] || die "Source directory does not exist: $SOURCE_DIR"

    mkdir -p "$BACKUP_DIR"

    TIMESTAMP="$(date '+%Y%m%d_%H%M%S')"
    BACKUP_FILE="${BACKUP_DIR}/backup_${TIMESTAMP}.tar.gz"

    log "Backing up: $SOURCE_DIR"
    log "Output:     $BACKUP_FILE"

    tar -czf "$BACKUP_FILE" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

    log "Backup created"

    find "$BACKUP_DIR" \
        -type f \
        -name 'backup_*.tar.gz' \
        -mtime "+$RETENTION_DAYS" \
        -delete

    log "Old backups cleaned"
}

main "$@"
```

Run:

```bash
chmod +x backup.sh

mkdir -p "$HOME/data"
echo "important data" > "$HOME/data/test.txt"

./backup.sh
```

Custom values:

```bash
SOURCE_DIR=/opt/myapp/data \
BACKUP_DIR=/backup/myapp \
RETENTION_DAYS=14 \
./backup.sh
```

### Changeable values

| Variable | Purpose | Example |
|---|---|---|
| `SOURCE_DIR` | what to back up | `/opt/myapp/data` |
| `BACKUP_DIR` | where archives go | `/backup/myapp` |
| `RETENTION_DAYS` | delete older archives | `14` |

---

## 20. Review before production

```bash
bash -n script.sh
shellcheck script.sh
bash -x script.sh   # while testing
```

Then test:
- success path
- missing file
- missing command
- permission failure
- empty variable
- unexpected input
- partial failure
- interrupted execution
