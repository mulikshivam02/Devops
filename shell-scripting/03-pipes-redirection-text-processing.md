# 03. Pipes, Redirection and Text Processing

## 1. Standard streams

Linux commands commonly use:

- stdin (`0`)
- stdout (`1`)
- stderr (`2`)

A script can redirect these streams.

---

## 2. Basic pipe `|`

```bash
ps aux | grep '[n]ginx'
```

Concept:

```text
ps aux
  |
  v
grep '[n]ginx'
```

The output of the first command becomes input to the next.

Bash documents a pipeline as commands connected by `|` or `|&`.

---

## 3. `|&`

```bash
command1 |& command2
```

This sends standard output and standard error through the pipeline.

Equivalent in Bash terms to:

```bash
command1 2>&1 | command2
```

---

## 4. Output redirection

Overwrite:

```bash
echo "hello" > output.txt
```

Append:

```bash
echo "another line" >> output.txt
```

---

## 5. Input redirection

```bash
wc -l < output.txt
```

---

## 6. stderr

Write only errors:

```bash
command 2> errors.log
```

Append:

```bash
command 2>> errors.log
```

---

## 7. stdout + stderr

Classic form:

```bash
command > output.log 2>&1
```

Order matters.

Bash processes redirections left-to-right.

A Bash-specific alternative:

```bash
command &> output.log
```

Append:

```bash
command &>> output.log
```

---

## 8. `tee`

Show output on screen and save it:

```bash
command | tee output.log
```

Append:

```bash
command | tee -a output.log
```

Example:

```bash
df -h | tee disk-report.txt
```

---

## 9. `grep`

Search:

```bash
grep "ERROR" app.log
```

Case-insensitive:

```bash
grep -i "error" app.log
```

Recursive:

```bash
grep -R "TODO" .
```

Show line numbers:

```bash
grep -n "ERROR" app.log
```

Invert match:

```bash
grep -v "INFO" app.log
```

Count:

```bash
grep -c "ERROR" app.log
```

---

## 10. `head` and `tail`

```bash
head file.txt
tail file.txt
```

First 20:

```bash
head -n 20 file.txt
```

Last 20:

```bash
tail -n 20 file.txt
```

Follow:

```bash
tail -f app.log
```

---

## 11. `wc`

```bash
wc -l file.txt
wc -w file.txt
wc -c file.txt
```

Count matching errors:

```bash
grep -i "error" app.log | wc -l
```

---

## 12. `sort`

```bash
sort names.txt
```

Numeric:

```bash
sort -n numbers.txt
```

Reverse:

```bash
sort -r names.txt
```

Sort by field:

```bash
sort -k2,2 names.txt
```

---

## 13. `uniq`

Usually sort first:

```bash
sort names.txt | uniq
```

Count:

```bash
sort names.txt | uniq -c
```

Most common:

```bash
sort names.txt | uniq -c | sort -nr
```

---

## 14. `cut`

Example CSV-like data:

```text
alice,dev,100
bob,ops,200
carol,dev,300
```

Second field:

```bash
cut -d',' -f2 users.csv
```

---

## 15. `tr`

Convert:

```bash
echo "hello" | tr '[:lower:]' '[:upper:]'
```

Replace characters:

```bash
echo "a-b-c" | tr '-' '_'
```

---

## 16. `awk`

Print field:

```bash
awk '{print $1}' file.txt
```

Field separator:

```bash
awk -F',' '{print $1, $3}' users.csv
```

Condition:

```bash
awk -F',' '$2 == "dev" {print $1}' users.csv
```

Sum numeric field:

```bash
awk -F',' '{sum += $3} END {print sum}' users.csv
```

---

## 17. `sed`

Replace first occurrence per line:

```bash
sed 's/old/new/' file.txt
```

Replace globally:

```bash
sed 's/old/new/g' file.txt
```

In-place:

```bash
sed -i 's/old/new/g' file.txt
```

For important files, make a backup or test the expression first.

---

## 18. `find`

Find files:

```bash
find /var/log -type f
```

By name:

```bash
find /var/log -type f -name '*.log'
```

By size:

```bash
find /var/log -type f -size +100M
```

By modification time:

```bash
find /var/log -type f -mtime +7
```

Execute a command:

```bash
find ./logs -type f -name '*.log' -exec gzip {} \;
```

---

## 19. `xargs`

Example:

```bash
printf '%s\n' a b c | xargs echo
```

For file names with spaces/newlines, prefer null-delimited workflows:

```bash
find . -type f -print0 | xargs -0 ls -l
```

Do not use `xargs` blindly with arbitrary untrusted text.

---

## 20. Multi-stage pipeline example

```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu |
head -n 11
```

Show only process names:

```bash
ps -eo comm,%cpu,%mem --sort=-%cpu |
head -n 11
```

Extract memory-heavy processes:

```bash
ps -eo comm,%mem --sort=-%mem |
head -n 11
```

---

## 21. Real log-analysis pipeline

```bash
grep -i "error" app.log |
awk '{print $1}' |
sort |
uniq -c |
sort -nr |
head -20
```

Typical interpretation:

```text
read log
  -> keep errors
  -> extract chosen field
  -> sort
  -> count duplicates
  -> sort by frequency
  -> top 20
```

Adjust `$1` after examining the actual log format.

---

## 22. Here document

```bash
cat > config.txt <<'EOF'
APP_NAME=myapp
APP_PORT=8080
APP_ENV=dev
EOF
```

`EOF` is just a delimiter. It can be another token, as long as the start/end markers match.

---

## 23. Here string

```bash
grep "error" <<< "error: application failed"
```

---

## 24. Complete example: pipeline report

Create:

```text
examples/03-pipeline-and-text-processing/
```

Files:

```text
app.log
pipeline-report.sh
```

`app.log`:

```text
2026-08-19 10:00:01 INFO server started
2026-08-19 10:00:05 ERROR database connection failed
2026-08-19 10:00:06 ERROR database connection failed
2026-08-19 10:00:07 WARN retrying
2026-08-19 10:00:08 ERROR timeout
2026-08-19 10:00:09 INFO connected
```

`pipeline-report.sh`:

```bash
#!/usr/bin/env bash

set -euo pipefail

LOG_FILE="${1:-app.log}"

if [[ ! -f "$LOG_FILE" ]]; then
    echo "File not found: $LOG_FILE" >&2
    exit 1
fi

echo "Total lines:"
wc -l < "$LOG_FILE"

echo
echo "ERROR count:"
grep -ic "error" "$LOG_FILE" || true

echo
echo "Top error messages:"
grep -i "error" "$LOG_FILE" |
    sort |
    uniq -c |
    sort -nr
```

Run:

```bash
cd examples/03-pipeline-and-text-processing
chmod +x pipeline-report.sh
./pipeline-report.sh
```

Use another file:

```bash
./pipeline-report.sh /path/to/application.log
```

### Change

```bash
LOG_FILE="${1:-app.log}"
```

`app.log` = default file.

---

## 25. Useful text-processing chain

```bash
cat app.log |
grep -i "error" |
awk '{print $NF}' |
sort |
uniq -c |
sort -nr |
head -10
```

Use `cat file | ...` only when it makes the pipeline clearer; many commands can read the file directly.
