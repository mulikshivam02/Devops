# 09. Shell / Linux / DevOps Command Cheat Sheet

## Navigation

```bash
pwd
ls -lah
cd /path
cd -
cd ..
```

## Files and directories

```bash
touch file.txt
mkdir -p dir/subdir
cp source dest
cp -r dir1 dir2
mv old new
rm file
rm -rf dir
```

Be extremely careful with `rm -rf`.

---

## File inspection

```bash
cat file
less file
head -n 20 file
tail -n 20 file
tail -f file
wc -l file
file file
stat file
```

---

## Search

```bash
grep "text" file
grep -Ri "text" .
find . -type f -name '*.log'
find /var/log -type f -size +100M
```

---

## Text processing

```bash
cut -d',' -f1 file.csv
awk '{print $1}' file
sed 's/old/new/g' file
sort file
uniq file
sort file | uniq -c | sort -nr
tr '[:lower:]' '[:upper:]'
xargs
```

---

## Pipes

```bash
command1 | command2
command1 | command2 | command3
command1 |& command2
```

---

## Redirection

```bash
cmd > output.txt
cmd >> output.txt
cmd < input.txt
cmd 2> error.txt
cmd 2>> error.txt
cmd > out.txt 2>&1
cmd &> all.log
cmd | tee output.txt
cmd | tee -a output.txt
```

---

## Processes

```bash
ps aux
ps -ef
pgrep -af nginx
pkill nginx
kill PID
kill -9 PID
jobs
fg
bg
```

---

## Resources

```bash
uptime
free -h
df -h
du -sh /path
top
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
nproc
lscpu
```

---

## Network

```bash
ip addr
ip route
ss -tulpn
ss -ltn
ping -c 4 HOST
curl -I https://example.com
curl -fsS URL
getent hosts example.com
```

---

## Services

```bash
systemctl status SERVICE
systemctl start SERVICE
systemctl stop SERVICE
systemctl restart SERVICE
systemctl is-active SERVICE
systemctl enable --now SERVICE
```

---

## Logs

```bash
journalctl -u SERVICE
journalctl -u SERVICE -n 100
journalctl -u SERVICE -f
journalctl -u SERVICE --since "1 hour ago"
```

---

## Permissions

```bash
chmod +x script.sh
chmod 755 script.sh
chmod 644 file.txt
chown USER:GROUP file
```

---

## Environment

```bash
env
printenv
echo "$PATH"
echo "$HOME"
echo "$USER"
export APP_ENV=dev
```

---

## Bash

```bash
bash --version
bash script.sh
bash -n script.sh
bash -x script.sh
source ./config.sh
```

---

## Script diagnostics

```bash
shellcheck script.sh
```

---

## Git

```bash
git status
git log --oneline --decorate -10
git diff
git pull --ff-only
git fetch origin
git branch
```

---

## Docker

```bash
docker ps
docker ps -a
docker images
docker logs CONTAINER
docker exec -it CONTAINER sh
docker inspect CONTAINER
docker build -t IMAGE:TAG .
docker run -d --name NAME -p HOST:CONTAINER IMAGE:TAG
```

---

## Kubernetes

```bash
kubectl get nodes
kubectl get pods
kubectl get pods -A
kubectl get svc
kubectl get deploy
kubectl describe pod POD
kubectl logs POD
kubectl exec -it POD -- sh
kubectl apply -f file.yaml
kubectl delete -f file.yaml
kubectl rollout status deployment/NAME
```

---

## Useful patterns

### Count

```bash
command | wc -l
```

### Filter

```bash
command | grep pattern
```

### Sort by numeric column

```bash
command | sort -nrk2
```

### Save and display

```bash
command | tee report.log
```

### Run only if successful

```bash
command1 && command2
```

### Run fallback

```bash
command1 || command2
```

### Use default variable

```bash
VALUE="${VALUE:-default}"
```

### Required variable

```bash
: "${VALUE:?VALUE is required}"
```

### Timestamp

```bash
date '+%Y-%m-%d %H:%M:%S'
```
