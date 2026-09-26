# Bash Scripting for DevOps — Hands-On Lab

Ten exercises that drill the exact habits from the interview guide. Work in a throwaway directory — everything here is safe to run locally.

```bash
mkdir -p ~/bash-lab && cd ~/bash-lab
```

---

## Exercise 1 — Feel strict mode catch a bug

**Goal:** See `set -euo pipefail` stop a script that would silently lie without it.

```bash
cat > strict.sh << 'SCRIPT'
#!/usr/bin/env bash
echo "pipefail OFF demo:"
false | echo "last command wins, pipeline reports success"
echo "exit code of pipeline: $?"
SCRIPT
bash strict.sh
```

**Expected output:** `exit code of pipeline: 0` — the pipeline "succeeded" even though `false` failed. Now add `set -o pipefail` after the shebang, rerun, and watch the exit code become 1.

**Why it matters:** `deploy.sh | tee deploy.log` reports success when the deploy failed, unless pipefail is set. Interviewers ask exactly this.

---

## Exercise 2 — Catch an unset variable before it hurts

**Goal:** Use `${VAR:?}` guards and `set -u` to fail fast on typos.

```bash
cat > guard.sh << 'SCRIPT'
#!/usr/bin/env bash
set -u
REGION="${AWS_REIGON:?AWS_REGION must be set}"
echo "Deploying to $REGION"
SCRIPT
bash guard.sh; echo "exit: $?"
```

**Expected output:** `bash: line 3: AWS_REIGON: AWS_REGION must be set` and `exit: 1`. Rename the variable to `AWS_REGION`, export it, and rerun to see it pass.

**Why it matters:** A misspelled variable deploying to the wrong region is a resume-generating event. Guards make typos loud.

---

## Exercise 3 — Quoting: break it, then fix it

**Goal:** Watch word splitting corrupt a filename with a space.

```bash
mkdir -p "my app" && touch "my app/config.yaml"
FILE="my app/config.yaml"
ls $FILE        # unquoted — breaks
ls "$FILE"      # quoted — works
```

**Expected output:** The unquoted `ls` errors with `ls: cannot access 'my': No such file or directory` plus `app/config.yaml`. The quoted version lists the file cleanly.

**Why it matters:** Unquoted variables in `rm -rf $DIR/*` are how wrong directories get deleted. This is muscle memory, not theory.

---

## Exercise 4 — The subshell trap in piped loops

**Goal:** Prove variables set inside a piped loop don't survive it.

```bash
seq 1 5 | while read -r n; do count=$((count+1)); done
echo "piped count: ${count:-unset}"
count=0
while read -r n; do count=$((count+1)); done < <(seq 1 5)
echo "redirected count: $count"
```

**Expected output:** `piped count: unset` then `redirected count: 5`. The piped loop ran in a subshell; the redirected one didn't.

**Why it matters:** Log-parsing and inventory scripts silently report zero when written with `cat file | while read`. Redirection keeps state.

---

## Exercise 5 — Parameter expansion one-liners

**Goal:** Replace if-blocks with `${VAR:-}` and `${VAR:?}`.

```bash
unset PORT;  echo "port: ${PORT:-8080}"
PORT="";     echo "port: ${PORT:-8080}"     # colon catches empty too
PORT="";     echo "port: ${PORT-8080}"      # no colon: empty stays empty
bash -c 'echo "${DB_HOST:?DB_HOST required}"'; echo "exit: $?"
```

**Expected output:** `port: 8080`, `port: 8080`, `port: ` (empty), then the error message and `exit: 1`.

**Why it matters:** One-liners replace whole validation blocks at the top of deploy scripts. Know the `:-` vs `-` difference cold.

---

## Exercise 6 — trap cleanup that actually fires

**Goal:** Build a script that cleans up even when it crashes.

```bash
cat > cleanup.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
TMPDIR="$(mktemp -d)"
cleanup() { echo "cleaning $TMPDIR"; rm -rf "$TMPDIR"; }
trap cleanup EXIT
echo "working in $TMPDIR"
false   # simulate a failed deploy step
SCRIPT
bash cleanup.sh; ls -d /tmp/tmp.* 2>/dev/null | head -3
```

**Expected output:** `cleaning /tmp/tmp.XXXXXX` printed before exit, and no leftover `tmp.*` dir from this run. Remove the `false` line and rerun — cleanup still fires on success.

**Why it matters:** Failed deploys leave lock files and half-written artifacts that poison the next run. Traps make cleanup unconditional.

---

## Exercise 7 — Debug a "works on my machine" script

**Goal:** Use `env -i` and `set -x` to find an environment-dependent bug.

```bash
cat > ci-bug.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
echo "Using tool at: $MY_TOOL"
"$MY_TOOL" --version
SCRIPT
export MY_TOOL=/usr/bin/git
bash ci-bug.sh            # works locally
env -i bash ci-bug.sh     # fails like CI does
```

**Expected output:** First run prints git's version. The `env -i` run fails with `ci-bug.sh: line 3: MY_TOOL: unbound variable` — exactly what CI sees with its near-empty environment.

**Why it matters:** Half of "works locally, fails in CI" is an inherited variable. `env -i` reproduces CI's environment in one command.

---

## Exercise 8 — Write an idempotent setup script

**Goal:** A script that's safe to run twice — the second run changes nothing.

```bash
cat > idempotent.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
mkdir -p ~/bash-lab/app
grep -q '^ENV=prod$' ~/bash-lab/app/.env 2>/dev/null \
  || echo 'ENV=prod' >> ~/bash-lab/app/.env
echo "done"
SCRIPT
bash idempotent.sh && bash idempotent.sh
cat ~/bash-lab/app/.env
```

**Expected output:** Both runs print `done`, and `.env` contains exactly one `ENV=prod` line — the second run appended nothing.

**Why it matters:** Cron retries and CI reruns execute your scripts repeatedly. `mkdir -p` and `grep -q`-before-append are the idempotency primitives.

---

## Exercise 9 — flock a script so two runs can't overlap

**Goal:** Prevent concurrent executions with an exclusive lock.

```bash
cat > locked.sh << 'SCRIPT'
#!/usr/bin/env bash
exec 200>/tmp/bash-lab.lock
flock -n 200 || { echo "another run is active, exiting"; exit 1; }
echo "got lock, working..."; sleep 5; echo "released"
SCRIPT
bash locked.sh & bash locked.sh; wait
```

**Expected output:** One process prints `got lock, working...` and finishes; the other prints `another run is active, exiting` and exits 1.

**Why it matters:** Overlapping deploys produce half-old half-new fleets. A lock turns a race condition into a clean skip.

---

## Exercise 10 — Assemble the deploy skeleton

**Goal:** Combine strict mode, guards, trap, logging, and health check into one shippable skeleton.

```bash
cat > deploy.sh << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

APP_DIR="${1:?usage: deploy.sh <app-dir>}"
VERSION="${VERSION:?VERSION must be set}"

log()  { echo "[$(date '+%F %T')] INFO  $*"; }
die()  { echo "[$(date '+%F %T')] ERROR $*" >&2; exit 1; }

rollback() { log "rolling back to previous version"; }
trap 'die "deploy failed"' ERR

log "deploying $VERSION to $APP_DIR"
# ... rsync / docker pull / kubectl set image here ...
curl -sf "http://localhost:8080/health" >/dev/null || { rollback; die "health check failed"; }
log "deploy of $VERSION healthy"
SCRIPT
bash -n deploy.sh && echo "syntax OK"
```

**Expected output:** `syntax OK` from `bash -n`. Run `VERSION=1.2.0 bash deploy.sh /tmp/fake` to watch it fail loudly at the health check and roll back — never silently.

**Why it matters:** This is the skeleton interviewers ask you to whiteboard: validate, deploy, verify, roll back. Now you've typed it once.
