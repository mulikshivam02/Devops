# 08. DevOps Shell Script Patterns

## 1. Environment validation

Create:

```bash
#!/usr/bin/env bash

set -u

required_commands=("docker" "curl" "git")

for cmd in "${required_commands[@]}"; do
    if command -v "$cmd" >/dev/null 2>&1; then
        echo "[OK] $cmd"
    else
        echo "[MISSING] $cmd" >&2
        exit 1
    fi
done

echo "Environment looks ready."
```

Change:

```bash
required_commands=("docker" "curl" "git")
```

to match the tools your deployment requires.

---

## 2. Docker wrapper

```bash
#!/usr/bin/env bash

set -euo pipefail

IMAGE="${IMAGE:-nginx:latest}"
CONTAINER_NAME="${CONTAINER_NAME:-my-nginx}"
PORT="${PORT:-8080}"

docker run -d \
    --name "$CONTAINER_NAME" \
    -p "${PORT}:80" \
    "$IMAGE"
```

Run:

```bash
IMAGE=nginx:1.28 CONTAINER_NAME=demo PORT=8080 ./run-container.sh
```

Change:
- `IMAGE`
- `CONTAINER_NAME`
- host `PORT`

Verify:

```bash
docker ps
curl http://127.0.0.1:8080
```

Cleanup:

```bash
docker rm -f demo
```

---

## 3. Git helper

```bash
#!/usr/bin/env bash

set -euo pipefail

BRANCH="${1:-main}"

git status
git fetch origin
git checkout "$BRANCH"
git pull --ff-only origin "$BRANCH"
```

Use carefully with local changes.

---

## 4. Build-test-deploy pattern

```bash
#!/usr/bin/env bash

set -euo pipefail

APP_DIR="${APP_DIR:-$PWD}"

cd "$APP_DIR"

echo "[1/3] Build"
./build.sh

echo "[2/3] Test"
./test.sh

echo "[3/3] Deploy"
./deploy.sh

echo "Deployment successful."
```

The key design idea is:

```text
build -> test -> deploy
```

Because `set -e` can stop the script on selected failures, it can prevent later stages from running after a failure. Still test the exact script behavior.

---

## 5. Deployment with rollback hook

A generic pattern:

```bash
#!/usr/bin/env bash

set -euo pipefail

release="$1"

deploy() {
    echo "Deploying $release"
    # deployment commands
}

rollback() {
    echo "Rollback requested"
    # rollback commands
}

trap rollback ERR

deploy "$release"

trap - ERR

echo "Deployment completed"
```

Do not copy this blindly into production. Complex error handling around `trap ERR` and Bash `errexit` needs deliberate testing.

---

## 6. Check application port

```bash
PORT="${1:-8080}"

if ss -ltn | awk '{print $4}' | grep -q ":${PORT}$"; then
    echo "Port $PORT is listening"
else
    echo "Port $PORT is not listening"
    exit 1
fi
```

A more robust implementation depends on the exact address format and desired IPv4/IPv6 semantics.

---

## 7. Cleanup old files

```bash
#!/usr/bin/env bash

set -euo pipefail

TARGET_DIR="${TARGET_DIR:-/tmp/myapp}"
AGE_DAYS="${AGE_DAYS:-7}"

[[ -d "$TARGET_DIR" ]] || exit 0

find "$TARGET_DIR" \
    -type f \
    -mtime "+$AGE_DAYS" \
    -delete
```

Dry-run first:

```bash
find "$TARGET_DIR" -type f -mtime "+$AGE_DAYS" -print
```

Only add `-delete` after verifying the selected files.

---

## 8. Deployment variables file

Create:

```text
.env
```

Example:

```bash
APP_NAME=myapp
APP_PORT=8080
APP_ENV=dev
IMAGE=myapp:latest
```

Then:

```bash
set -a
source ./.env
set +a

echo "$APP_NAME"
```

Only source trusted files. Do not source arbitrary downloaded/user-provided files.

---

## 9. Kubernetes helper pattern

Example:

```bash
#!/usr/bin/env bash

set -euo pipefail

NAMESPACE="${NAMESPACE:-default}"
MANIFEST="${MANIFEST:-deployment.yaml}"

command -v kubectl >/dev/null 2>&1 || {
    echo "kubectl is required" >&2
    exit 1
}

kubectl apply -n "$NAMESPACE" -f "$MANIFEST"
kubectl rollout status -n "$NAMESPACE" deployment/"${APP_NAME:-myapp}"
```

Change:
- `NAMESPACE`
- `MANIFEST`
- `APP_NAME`

Use only when the manifest and deployment name are known.

---

## 10. Docker cleanup helper

Preview first:

```bash
docker ps -a
```

Then script:

```bash
#!/usr/bin/env bash

set -euo pipefail

docker container prune -f
docker image prune -f
```

Do not place aggressive cleanup commands into unattended servers until you understand the impact.

---

## 11. Common DevOps script flow

A very reusable structure:

```text
1. Parse arguments
2. Set defaults
3. Validate inputs
4. Check dependencies
5. Check permissions
6. Run the operation
7. Verify the result
8. Log outcome
9. Clean temporary state
10. Return meaningful exit status
```

---

## 12. Complete example: deployment wrapper

Create:

```text
examples/08-deploy-app/deploy.sh
```

```bash
#!/usr/bin/env bash

set -euo pipefail

APP_NAME="${APP_NAME:-myapp}"
IMAGE="${IMAGE:-nginx:latest}"
CONTAINER_NAME="${CONTAINER_NAME:-$APP_NAME}"
HOST_PORT="${HOST_PORT:-8080}"
CONTAINER_PORT="${CONTAINER_PORT:-80}"

log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

require_command() {
    command -v "$1" >/dev/null 2>&1 || {
        echo "Missing command: $1" >&2
        exit 1
    }
}

main() {
    require_command docker
    require_command curl

    log "Deploying $APP_NAME"

    if docker ps -a --format '{{.Names}}' |
        grep -Fxq "$CONTAINER_NAME"; then
        log "Removing previous container"
        docker rm -f "$CONTAINER_NAME"
    fi

    log "Starting container"

    docker run -d \
        --name "$CONTAINER_NAME" \
        -p "${HOST_PORT}:${CONTAINER_PORT}" \
        "$IMAGE"

    log "Waiting for application"

    for _ in {1..10}; do
        if curl -fsS --max-time 2 "http://127.0.0.1:${HOST_PORT}/" >/dev/null; then
            log "Application is responding"
            return 0
        fi

        sleep 2
    done

    echo "Application did not become ready" >&2
    docker logs "$CONTAINER_NAME" >&2 || true
    return 1
}

main "$@"
```

Run:

```bash
chmod +x deploy.sh

IMAGE=nginx:latest \
APP_NAME=myapp \
CONTAINER_NAME=myapp \
HOST_PORT=8080 \
CONTAINER_PORT=80 \
./deploy.sh
```

Verify:

```bash
docker ps
curl http://127.0.0.1:8080/
```

Cleanup:

```bash
docker rm -f myapp
```

### What to modify

```text
IMAGE          -> Docker image
APP_NAME       -> logical application name
CONTAINER_NAME -> Docker container name
HOST_PORT      -> port on your machine
CONTAINER_PORT -> port exposed by application inside container
```

This is a template; readiness for real applications is usually better handled through an application-specific health endpoint rather than simply checking `/`.
