# 12-Factor App Methodology — Hands-On Lab

*Turn the twelve factors into muscle memory. You'll take a small Python app that violates half the factors and fix it one factor at a time. Work in `/tmp/twelve-factor-lab`.*

**Setup:**

```bash
mkdir -p /tmp/twelve-factor-lab && cd /tmp/twelve-factor-lab
# the "before" app: config hardcoded, deps unpinned, state in memory, logs to file
cat > app.py <<'EOF'
import sqlite3, logging
logging.basicConfig(filename="app.log", level=logging.INFO)
DB_PATH = "/data/app.db"          # hardcoded path — factor 3 violation
PORT = 8080                       # hardcoded port — factor 7 violation
SESSIONS = {}                     # in-memory sessions — factor 6 violation
logging.info("app starting")
print("listening on 8080")
EOF
cat app.py
```

Expected output: you can see the three violations sitting in plain sight — that's the point.

---

## Exercise 1 — Spot the violations

**Goal:** Map each smell in `app.py` to its factor before fixing anything.

**Commands:**

```bash
grep -n "DB_PATH\|PORT\|SESSIONS\|app.log" app.py
```

**Expected output:** Four hits, one per smell: hardcoded DB path (factor 3, config), hardcoded port (factor 7, port binding), in-memory sessions (factor 6, stateless processes), log file (factor 11, logs).

**Why it matters:** Interviewers love "what's wrong with this app?" questions. Training your eye on a real file beats memorizing the list.

---

## Exercise 2 — Externalize config into the environment

**Goal:** Move the DB path and port out of code and into env vars (factor 3).

**Commands:**

```bash
cat > app.py <<'EOF'
import os, sqlite3, logging
logging.basicConfig(level=logging.INFO)
DB_PATH = os.environ["DB_PATH"]        # config, not code
PORT = int(os.environ.get("PORT", "8080"))
SESSIONS = {}
logging.info("app starting on port %s", PORT)
print(f"listening on {PORT}")
EOF
DB_PATH=/tmp/dev.db PORT=9000 python3 app.py
echo "---"
DB_PATH=/tmp/prod.db PORT=8080 python3 app.py
```

**Expected output:** Same file runs twice with different config — no code change between "environments."

**Why it matters:** This is the factor-three deploy model in miniature: one artifact, many environments, config supplied at runtime.

---

## Exercise 3 — Declare and pin dependencies

**Goal:** Replace implicit system packages with an explicit, pinned manifest (factor 2).

**Commands:**

```bash
cat > requirements.txt <<'EOF'
flask==3.0.3
gunicorn==22.0.0
EOF
cat > Dockerfile <<'EOF'
FROM python:3.12-slim-bookworm
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python3", "app.py"]
EOF
cat requirements.txt
```

**Expected output:** Every dependency named with an exact version — rebuild this image in a year and you get the same app.

**Why it matters:** Unpinned deps are how "the base image updated and everything broke" happens. Explicit and isolated means reproducible from scratch.

---

## Exercise 4 — Swap a backing service with one variable

**Goal:** Prove the app doesn't care which database it talks to (factor 4).

**Commands:**

```bash
# dev: sqlite file. prod: postgres URL. Same code, same image.
DB_PATH=/tmp/dev.db PORT=9000 python3 app.py    # dev backing service
echo "---"
DB_PATH="postgresql://app:secret@db.internal:5432/appdb" PORT=8080 python3 app.py  # prod backing service
```

**Expected output:** Both invocations start cleanly — the app treats the database as an attached resource, a URL in config.

**Why it matters:** Vendor lock-in is a code smell you can grep for. If switching databases requires a code change, you're coupled to a backing service.

---

## Exercise 5 — Separate build, release, and run

**Goal:** Build once, release by pairing the image with config, run unchanged (factor 5).

**Commands:**

```bash
# BUILD: once, from code
docker build -t twelve-factor-app:1.0.0 .
# RELEASE: build + config = uniquely identified release
docker tag twelve-factor-app:1.0.0 twelve-factor-app:release-2026-09-30
# RUN: the release, executed with environment-specific config
docker run -d --name app-staging -e DB_PATH=/tmp/staging.db -e PORT=8080 twelve-factor-app:release-2026-09-30
docker run -d --name app-prod    -e DB_PATH=/tmp/prod.db    -e PORT=8080 twelve-factor-app:release-2026-09-30
docker ps --format '{{.Names}} -> {{.Image}}'
```

**Expected output:** Two containers from the same immutable release, differing only in config. Rollback = run the previous release tag.

**Why it matters:** This is the exact shape of every CI/CD pipeline: build the image once, promote it through environments, never patch a running release.

---

## Exercise 6 — Kill the state: externalize sessions

**Goal:** Move sessions out of the process so any instance can serve any request (factor 6).

**Commands:**

```bash
# the anti-pattern: two "instances" with separate in-memory session stores
python3 - <<'EOF'
# simulate: user logs in on instance A, request lands on instance B
instance_a_sessions = {"user42": "logged-in"}
instance_b_sessions = {}
print("request for user42 hits instance B:", instance_b_sessions.get("user42", "SESSION LOST"))
# fixed: both instances read from one shared store (Redis in real life)
shared_store = {"user42": "logged-in"}
print("with shared store, instance B sees:", shared_store.get("user42"))
EOF
```

**Expected output:** "SESSION LOST" for the in-memory version, "logged-in" with the shared store.

**Why it matters:** This is the demo that kills sticky sessions in an interview. If any instance can serve any request, you can scale, restart, and roll without losing users.

---

## Exercise 7 — Bind your own port

**Goal:** Make the app self-contained: it listens on `$PORT`, no web server injected (factor 7).

**Commands:**

```bash
# the app owns its port — the platform just routes to it
PORT=5001 python3 app.py & APP_PID=$!
sleep 1
curl -s -o /dev/null -w "responded on 5001: %{http_code}\n" http://localhost:5001/ || echo "(no HTTP handler yet — the point is the port is OURS)"
kill $APP_PID 2>/dev/null
# in Kubernetes this becomes: containerPort: 5001, Service targets it, Ingress routes to the Service
kubectl run demo --image=twelve-factor-app:1.0.0 --port=5001 --dry-run=client -o yaml | grep -A2 ports || true
```

**Expected output:** The app starts on whichever port the environment assigns — no container or platform config needed inside the app.

**Why it matters:** Port binding is why "what port does it listen on" is a deploy-time question. The app exports a service; the platform routes to it.

---

## Exercise 8 — Shut down gracefully

**Goal:** Handle SIGTERM like a disposable process: drain, then exit (factor 9).

**Commands:**

```bash
cat > graceful.py <<'EOF'
import signal, sys, time
running = True
def handle_term(signum, frame):
    global running
    print("SIGTERM received: draining in-flight requests...", flush=True)
    time.sleep(2)  # finish current work
    print("drained, exiting cleanly", flush=True)
    sys.exit(0)
signal.signal(signal.SIGTERM, handle_term)
print("ready", flush=True)
while running:
    time.sleep(1)
EOF
python3 graceful.py & GPID=$!
sleep 1
kill -TERM $GPID
wait $GPID
```

**Expected output:** "SIGTERM received: draining in-flight requests..." → "drained, exiting cleanly" — no dropped work, no SIGKILL.

**Why it matters:** This two-line handler is the difference between invisible rolling updates and user-facing 502s. It's the highest-leverage code in any deploy pipeline.

---

## Exercise 9 — Stream logs to stdout, not files

**Goal:** Replace the log file with an event stream the platform can route (factor 11).

**Commands:**

```bash
# before: logs vanish into a file inside the container (dies with it)
ls -la app.log 2>/dev/null && echo "log file exists — invisible to the platform"
# after: stream to stdout, let the platform aggregate
python3 - <<'EOF'
import logging, sys
logging.basicConfig(stream=sys.stdout, level=logging.INFO,
                    format='{"level":"%(levelname)s","msg":"%(message)s"}')
logging.info("payment processed")
logging.warning("retrying backing service")
EOF
```

**Expected output:** Structured JSON lines on stdout — exactly what `docker logs`, `kubectl logs`, or a log shipper consumes.

**Why it matters:** In a fleet of disposable containers, a log file is write-only memory. Stdout streaming is what makes centralized logging possible.
