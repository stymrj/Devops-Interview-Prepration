# 12-Factor App Methodology — Cheat Sheet

## The twelve factors, one line each

| # | Factor | One line |
|---|--------|----------|
| I | Codebase | One repo in version control, many deploys from it |
| II | Dependencies | Declare explicitly in a manifest, isolate from the system |
| III | Config | Store in the environment, never in code |
| IV | Backing services | Treat as attached resources, swappable via URL |
| V | Build, release, run | Strictly separated stages; releases immutable |
| VI | Processes | Stateless, share-nothing; state lives in backing services |
| VII | Port binding | App is self-contained, binds its own port, exports HTTP |
| VIII | Concurrency | Scale out via the process model, one process type per concern |
| IX | Disposability | Fast startup, graceful SIGTERM shutdown |
| X | Dev/prod parity | Keep time, personnel, and tooling gaps small |
| XI | Logs | Event streams to stdout; the platform routes them |
| XII | Admin processes | One-off tasks run in the identical environment |

## Config in the environment

```bash
# anything that varies between deploys is config
DATABASE_URL="postgres://app@db.internal:5432/appdb"
REDIS_URL="redis://cache.internal:6379/0"
LOG_LEVEL=warn
PORT=8080
# same image everywhere, only env changes
docker run -e DATABASE_URL="$DATABASE_URL" -e PORT=8080 myapp:1.4.2
```

Config = varies between deploys. Everything else = code.

## Dependencies: declared and isolated

```dockerfile
# pin the base image AND the packages
FROM python:3.12-slim-bookworm
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

- Manifest: `requirements.txt`, `package.json`, `go.mod` — exact versions
- Isolation: virtualenv or container — never system-wide installs
- Smell: `FROM x:latest`, unpinned `apt-get install`, system tools assumed present

## Build, release, run

```
BUILD   : code  -> executable bundle (image)        [once]
RELEASE : build + config -> uniquely identified unit [cheap, traceable]
RUN     : release executed in an environment         [unchanged]
```

```bash
docker build -t myapp:1.4.2 .                       # build once
docker tag myapp:1.4.2 myapp:release-2026-09-30     # release = build + config
docker run -e DATABASE_URL="$PROD_DB" myapp:release-2026-09-30
# rollback = run the previous release tag. Never patch a running release.
```

## Stateless processes

- Session in memory → lost on restart/scale. Session in Redis → any instance serves any request.
- Uploads to local disk → lost. Uploads to S3 → survive anything.
- Sticky sessions = the trap. Load balancer stays dumb, processes stay disposable.

## Port binding

```bash
PORT=${PORT:-8080}
./myapp --port "$PORT"     # app owns its port; platform routes to it
```

Kubernetes: container binds port → Service targets it → Ingress routes. No web server injected.

## Disposability

```python
import signal, sys
def handle_term(signum, frame):
    stop_accepting_new_requests()
    drain_in_flight(timeout=25)
    sys.exit(0)
signal.signal(signal.SIGTERM, handle_term)
```

- Fast startup: seconds, not minutes — slow boot kills autoscaling
- Graceful shutdown: SIGTERM → drain → exit inside `terminationGracePeriodSeconds`
- Ignoring SIGTERM then eating SIGKILL = dropped requests on every deploy

## Dev/prod parity gaps

| Gap | Keep small by |
|-----|---------------|
| Time | Deploy within hours of writing code — continuous deployment |
| Personnel | Developers deploy, not a separate release team |
| Tools | Same DB engine, same base images, staging mirrors prod |

## Logs as event streams

```python
import logging, sys
logging.basicConfig(stream=sys.stdout, level=logging.INFO,
    format='{"level":"%(levelname)s","msg":"%(message)s"}')
```

- Never log files, never manage rotation in-app
- Platform captures stdout → ships to ELK / Loki / CloudWatch
- Log file inside a container = write-only memory, dies with the container

## Admin processes

- Migrations, backfills, one-off scripts = run in the identical environment
- Kubernetes: a Job or `kubectl run` with the same image and config
- Never run admin tasks from a laptop with different credentials

## Traps

- "Separate branch per environment" — violates factor I (one codebase)
- Hardcoded service endpoints — violates factor IV (backing services as attached resources)
- SSH-ing into a container to hotfix — violates factor V (immutable releases)
- "Just use sticky sessions" — violates factor VI (stateless processes)
- **Secrets in env vars** — factor III's letter vs its spirit. Env vars leak via `/proc`, crash dumps, CI logs. Plain config in env, secrets in a secrets manager (Vault, AWS Secrets Manager, K8s Secrets).

## Interview one-liners

- "One artifact, many environments — config is what varies between deploys."
- "If swapping your database requires a code change, you're coupled to a backing service."
- "What you tested is what you shipped — releases are immutable, rollbacks are just running the old one."
- "Sticky sessions make the load balancer stateful and every scale-down a data loss event."
- "12-factor says no secrets in code; the industry evolved the mechanism to secrets managers."
