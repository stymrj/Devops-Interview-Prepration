# DevOps and SRE Fundamentals — Hands-On Lab

*Practice the SRE habits: spotting toil, killing it with automation, measuring reliability. Safe to break things — you'll work in `/tmp/devops-sre-lab`.*

**Setup:**

```bash
mkdir -p /tmp/devops-sre-lab && cd /tmp/devops-sre-lab
# a fake "production service" you'll operate all lab long
cat > service.sh <<'EOF'
#!/bin/bash
echo "service started at $(date)" >> service.log
sleep 300
EOF
chmod +x service.sh
git init -q -b main 2>/dev/null; git add -A 2>/dev/null; git commit -qm "day 0: manual ops" 2>/dev/null
```

---

## Exercise 1 — Spot the toil

**Goal:** Learn the 5-question toil test by auditing your own week.

**Commands:**

```bash
cat > toil-audit.md <<'EOF'
| Task | Manual? | Repetitive? | Automatable? | Tactical? | Scales w/ growth? | TOIL? |
|------|---------|-------------|--------------|-----------|-------------------|-------|
| rotate logs on 8 servers | yes | weekly | yes | yes | yes | YES |
| design new deploy pipeline | no | once | n/a | no (strategic) | no | NO |
| restart stuck sidecar | yes | 3x/week | yes | yes | yes | YES |
EOF
cat toil-audit.md
```

**Expected output:** The table renders; at least the log rotation and sidecar restarts score YES on all five tests.

**Why it matters:** Interviewers love "give me an example of toil" — this gives you a real one from your own work, classified with Google's exact criteria instead of vibes.

---

## Exercise 2 — Kill one toil task permanently

**Goal:** Automate the log-rotation toil from Exercise 1.

**Commands:**

```bash
cat > rotate-logs.sh <<'EOF'
#!/bin/bash
# replaces the manual Monday log rotation
find /tmp/devops-sre-lab -name "*.log" -mtime +7 -exec gzip {} \;
find /tmp/devops-sre-lab -name "*.log.gz" -mtime +30 -delete
echo "$(date): rotated, disk now $(df -h /tmp | awk 'NR==2{print $5}') full" >> rotation.log
EOF
chmod +x rotate-logs.sh
./rotate-logs.sh && cat rotation.log
# schedule it: runs every Monday 06:00, no human involved
(crontab -l 2>/dev/null; echo "0 6 * * 1 /tmp/devops-sre-lab/rotate-logs.sh") | crontab -
crontab -l | grep rotate-logs
```

**Expected output:** `rotation.log` shows a timestamped run; `crontab -l` shows the Monday 06:00 entry.

**Why it matters:** This is the SRE move in miniature — toil eliminated permanently, not done faster. "I automated X" beats "I did X quickly" in every interview.

---

## Exercise 3 — Build a feedback loop: health checks

**Goal:** Give your service the observability "you build it, you run it" demands.

**Commands:**

```bash
./service.sh & echo $! > service.pid
cat > healthcheck.sh <<'EOF'
#!/bin/bash
if kill -0 "$(cat /tmp/devops-sre-lab/service.pid)" 2>/dev/null; then
  echo "$(date): OK - service alive"
else
  echo "$(date): CRITICAL - service dead, paging on-call" | tee -a alerts.log
fi
EOF
chmod +x healthcheck.sh
./healthcheck.sh
kill "$(cat service.pid)" && sleep 1 && ./healthcheck.sh
```

**Expected output:** First run prints `OK - service alive`; after the kill it prints `CRITICAL` and appends to `alerts.log`.

**Why it matters:** Feedback loops are the DevOps superpower — this 10-line script is the ancestor of every Prometheus alert you'll ever write. Detection must be automatic, not "a user complained."

---

## Exercise 4 — Measure deployment frequency from git history

**Goal:** Compute DORA's first metric with zero tooling.

**Commands:**

```bash
cd /tmp/devops-sre-lab
# simulate a month of small-batch deploys
for i in 1 2 3 4 5 6; do echo "change $i" >> app.txt; git add app.txt; git commit -qm "deploy: change $i"; done
echo "deploys in repo history: $(git log --oneline | wc -l)"
git log --since="30 days ago" --oneline | wc -l
```

**Expected output:** A count of deploys (commits) in the window — your deployment frequency, no dashboard required.

**Why it matters:** Proves the Q11 claim: you can start measuring DORA metrics today with git log and honest timestamps. Tooling comes later.

---

## Exercise 5 — Spend an error budget

**Goal:** Turn "99.9% SLO" into a number you can decide with.

**Commands:**

```bash
cat > error-budget.sh <<'EOF'
#!/bin/bash
# usage: error-budget.sh <slo-percent> <total-requests> <failed-requests>
SLO=$1; TOTAL=$2; FAILED=$3
ALLOWED=$(awk -v s="$SLO" -v t="$TOTAL" 'BEGIN{printf "%d", t*(100-s)/100}')
REMAINING=$((ALLOWED - FAILED))
echo "SLO ${SLO}% of ${TOTAL} reqs -> allowed failures: ${ALLOWED}, remaining budget: ${REMAINING}"
[ "$REMAINING" -gt 0 ] && echo "DECISION: budget remains, keep shipping" || echo "DECISION: budget burned, freeze deploys, invest in reliability"
EOF
chmod +x error-budget.sh
./error-budget.sh 99.9 1000000 500
./error-budget.sh 99.9 1000000 1200
```

**Expected output:** First run: 1000 allowed, 500 remaining → "keep shipping". Second: −200 remaining → "freeze deploys".

**Why it matters:** This is Q10 made executable — error budgets turn the dev-vs-ops argument into arithmetic both sides can see.

---

## Exercise 6 — Feel why small batches win

**Goal:** Compare one big-bang change vs three small ones.

**Commands:**

```bash
cd /tmp/devops-sre-lab
git checkout -qb bigbang -q
printf 'line%.0s\n' {1..50} >> big.txt && git add big.txt && git commit -qm "huge change: 50 lines at once"
git diff --stat HEAD~1 | tail -1
git checkout -q main
for i in 1 2 3; do echo "small change $i" >> small.txt; git add small.txt; git commit -qm "small change $i"; done
git log --oneline -3 && git diff --stat HEAD~3 | tail -1
```

**Expected output:** The big-bang commit shows one 50-line diff; the three small commits each show a tiny diff with a clear message.

**Why it matters:** Makes Q9 tangible — when the big one breaks, you bisect 50 lines; when a small one breaks, the commit message tells you what happened.

---

## Exercise 7 — Run chaos-lite: measure your MTTR

**Goal:** Practice incident response and time your recovery.

**Commands:**

```bash
cd /tmp/devops-sre-lab
./service.sh & echo $! > service.pid
START=$(date +%s)
kill -9 "$(cat service.pid)"   # the "incident"
./healthcheck.sh                # detection
./service.sh & echo $! > service.pid  # mitigation: restart
./healthcheck.sh                # verification
END=$(date +%s); echo "MTTR: $((END-START)) seconds (manual)"
```

**Expected output:** `MTTR: <a few> seconds (manual)` — detection to recovery, timed.

**Why it matters:** MTTR is DORA's fourth metric and the one juniors never measure. Timing yourself once makes "we recovered in 4 minutes" a real interview answer instead of a guess.

---

## Exercise 8 — Automate the recovery

**Goal:** Turn the manual restart from Exercise 7 into a watchdog.

**Commands:**

```bash
cat > watchdog.sh <<'EOF'
#!/bin/bash
# poor man's supervisor: restart the service if the health check fails
cd /tmp/devops-sre-lab
if ! ./healthcheck.sh | grep -q OK; then
  echo "$(date): watchdog restarting service" >> alerts.log
  ./service.sh & echo $! > service.pid
fi
EOF
chmod +x watchdog.sh
kill -9 "$(cat service.pid)"; sleep 1
./watchdog.sh && ./healthcheck.sh
```

**Expected output:** `alerts.log` shows the watchdog restart; the health check prints `OK` without you touching the service.

**Why it matters:** This is the toil loop closing: manual recovery (Ex 7) → automated recovery (Ex 8). Every systemd `Restart=always` and Kubernetes liveness probe is this script, grown up.

---

## Exercise 9 — Write the runbook

**Goal:** Produce the one-page artifact "you build it, you run it" requires.

**Commands:**

```bash
cat > RUNBOOK-service.md <<'EOF'
# Runbook: service.sh
## Symptoms
- healthcheck.sh reports CRITICAL; alerts.log has entries
## Diagnosis
1. `./healthcheck.sh` — confirm process dead vs wedged
2. `tail -20 service.log` — last words before death
3. `df -h /tmp`, `free -m` — disk/memory pressure?
## Fix
1. `./service.sh & echo $! > service.pid` (or rely on watchdog.sh)
2. Re-run healthcheck.sh until OK
## Escalate when
- restarts more than 3x/hour -> page service owner, likely a code bug not an ops issue
## Last reviewed
2026-09-29
EOF
cat RUNBOOK-service.md
```

**Expected output:** A complete one-page runbook file in the lab directory.

**Why it matters:** Runbooks are how DevOps teams share operational knowledge instead of hoarding it — and "show me a runbook you've written" is a real interview ask. Now you have one.

---

*Cleanup: `crontab -l | grep -v rotate-logs | crontab -` removes the lab cron entry; `rm -rf /tmp/devops-sre-lab` removes the lab.*
