# DORA Metrics and Platform Engineering — Hands-On Lab

*Turn DORA from buzzwords into numbers you computed yourself. You'll measure all four metrics from a simulated git history and deploy log, then build a mini platform "golden path" template. Work in `/tmp/dora-lab`.*

**Setup:**

```bash
mkdir -p /tmp/dora-lab && cd /tmp/dora-lab
# simulate 90 days of deploys: date,deploy_id,commit_sha,lead_hours,result,restore_minutes
python3 - <<'EOF'
import random, datetime
random.seed(13)
start = datetime.date(2026, 7, 2)
with open("deploys.csv", "w") as f:
    f.write("date,deploy_id,commit_sha,lead_hours,result,restore_minutes\n")
    d = start
    n = 0
    while d < datetime.date(2026, 9, 30):
        # ~3 deploys/week with some gaps
        if random.random() < 0.45:
            n += 1
            lead = round(random.gauss(50, 25), 1)
            failed = random.random() < 0.18
            restore = round(random.gauss(90, 40), 1) if failed else 0
            f.write(f"{d},dep-{n:03d},abc{n:04d},{max(1,lead)},{'fail' if failed else 'ok'},{max(5,restore)}\n")
        d += datetime.timedelta(days=1)
print("deploys.csv ready")
EOF
wc -l deploys.csv && head -4 deploys.csv
```

Expected output: ~40 rows of deploy data with realistic noise.

---

## Exercise 1 — Compute deployment frequency

**Goal:** Measure how often you actually ship to prod.

**Commands:**

```bash
total=$(($(wc -l < deploys.csv) - 1))
echo "Deploys in 90 days: $total"
python3 -c "print(f'Deploys per week: {$total/12.86:.1f}')"
```

**Expected output:** ~40 deploys in 90 days, roughly 3 per week — that's "medium" on the DORA scale, bordering high.

**Why it matters:** Frequency is the rawest signal of delivery health. Interviewers will ask "how often did you deploy?" — this is how you answer with a number instead of "pretty often."

---

## Exercise 2 — Compute lead time for changes

**Goal:** Find the median time from commit to production.

**Commands:**

```bash
python3 - <<'EOF'
import csv, statistics
with open("deploys.csv") as f:
    leads = [float(r["lead_hours"]) for r in csv.DictReader(f)]
print(f"Median lead time: {statistics.median(leads):.1f} hours")
print(f"P90 lead time: {sorted(leads)[int(len(leads)*0.9)]:.1f} hours")
EOF
```

**Expected output:** Median around 50 hours (~2 days), P90 much higher — the long tail is where the pain lives.

**Why it matters:** DORA uses medians because one month-long branch shouldn't define your team. Knowing your P90 tells you where the process breaks down.

---

## Exercise 3 — Compute change failure rate

**Goal:** Measure what fraction of deploys hurt users.

**Commands:**

```bash
python3 - <<'EOF'
import csv
with open("deploys.csv") as f:
    rows = list(csv.DictReader(f))
fails = sum(1 for r in rows if r["result"] == "fail")
print(f"Change failure rate: {100*fails/len(rows):.1f}% ({fails}/{len(rows)})")
EOF
```

**Expected output:** ~18% — that's "medium" (elite is under 15%), so this team is close but not elite.

**Why it matters:** This is the stability counterweight to frequency. A team shipping daily with a 40% failure rate isn't elite — they're reckless. Interviewers watch for this pairing.

---

## Exercise 4 — Compute MTTR

**Goal:** Measure mean time to restore service after a failure.

**Commands:**

```bash
python3 - <<'EOF'
import csv, statistics
with open("deploys.csv") as f:
    restores = [float(r["restore_minutes"]) for r in csv.DictReader(f) if r["result"] == "fail"]
print(f"Failed deploys: {len(restores)}")
print(f"Mean time to restore: {statistics.mean(restores):.0f} minutes")
EOF
```

**Expected output:** MTTR around 90 minutes — "high" band (under a day), not elite (under an hour).

**Why it matters:** MTTR is the metric users actually feel. Every minute of restore time is a minute of someone else's checkout page failing.

---

## Exercise 5 — Classify your team on the DORA scale

**Goal:** Put all four numbers together and read the verdict honestly.

**Commands:**

```bash
python3 - <<'EOF'
import csv, statistics
with open("deploys.csv") as f:
    rows = list(csv.DictReader(f))
freq = len(rows) / 12.86
lead = statistics.median(float(r["lead_hours"]) for r in rows)
cfr = 100 * sum(1 for r in rows if r["result"] == "fail") / len(rows)
mttr = statistics.mean(float(r["restore_minutes"]) for r in rows if r["result"] == "fail")
print(f"Deploy frequency : {freq:.1f}/week  -> {'high' if freq >= 7 else 'medium'}")
print(f"Lead time        : {lead:.0f}h        -> {'high' if lead < 168 else 'medium'}")
print(f"Change fail rate : {cfr:.0f}%         -> {'high' if cfr < 15 else 'medium'}")
print(f"MTTR             : {mttr:.0f} min      -> {'high' if mttr < 1440 else 'medium'}")
EOF
```

**Expected output:** A four-line scorecard, mostly "medium/high" — a solid team with clear next steps.

**Why it matters:** This is the exact exercise to describe in interviews: "I measured our DORA baseline, found lead time was the bottleneck, and here's what we changed."

---

## Exercise 6 — Find the bottleneck

**Goal:** Identify which metric is holding the team back the most.

**Commands:**

```bash
# which deploys had the worst lead times? correlate with day of week
python3 - <<'EOF'
import csv, datetime
from collections import defaultdict
with open("deploys.csv") as f:
    by_day = defaultdict(list)
    for r in csv.DictReader(f):
        dow = datetime.date.fromisoformat(r["date"]).strftime("%a")
        by_day[dow].append(float(r["lead_hours"]))
for d in ["Mon","Tue","Wed","Thu","Fri","Sat","Sun"]:
    vals = by_day.get(d, [])
    if vals: print(f"{d}: avg lead {sum(vals)/len(vals):.0f}h over {len(vals)} deploys")
EOF
```

**Expected output:** Deploys late in the week show longer lead times — the classic "don't deploy on Friday" freeze pattern stretching changes into Monday.

**Why it matters:** DORA numbers without diagnosis are trivia. The bottleneck analysis is what turns a dashboard into a plan — and it's what interviewers actually want to hear.

---

## Exercise 7 — Simulate one improvement

**Goal:** See how a one-click rollback policy changes the scorecard.

**Commands:**

```bash
python3 - <<'EOF'
import csv, statistics
with open("deploys.csv") as f:
    rows = list(csv.DictReader(f))
before = statistics.mean(float(r["restore_minutes"]) for r in rows if r["result"] == "fail")
# simulate: one-click rollback cuts every restore to under 30 min
after = statistics.mean(min(30, float(r["restore_minutes"])) for r in rows if r["result"] == "fail")
print(f"MTTR before one-click rollback: {before:.0f} min")
print(f"MTTR after one-click rollback:  {after:.0f} min  -> elite band (<60 min)")
EOF
```

**Expected output:** MTTR drops from ~90 to ~25 minutes — one automation moves a whole metric band.

**Why it matters:** This is the 90-day-plan answer in miniature: the highest-leverage change is usually making recovery trivial, not making deploys perfect.

---

## Exercise 8 — Build a golden path service template

**Goal:** Create the smallest possible "paved road" — a new-service scaffold with CI, Dockerfile, and health checks pre-wired.

**Commands:**

```bash
mkdir -p golden-path/myservice && cd golden-path/myservice
cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "server.js"]
EOF
cat > .github/workflows/deploy.yml <<'EOF'
name: deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t myservice:${{ github.sha }} .
      - run: echo "deploy ${{ github.sha }} to prod"
EOF
find . -type f | sort
```

**Expected output:** A scaffolded service with Dockerfile and deploy pipeline — any developer clones this and ships on day one.

**Why it matters:** Golden paths are platform engineering made concrete. The platform team's product is exactly this: the boring parts, pre-decided.

---

## Exercise 9 — Write the platform's "paved road" README

**Goal:** Document the golden path like a product — what it gives you, what it assumes, how to go off-road.

**Commands:**

```bash
cat > PLATFORM.md <<'EOF'
# Paved Road: Node.js Service

You get for free: CI build, prod deploy on merge, Prometheus metrics,
structured logs, auto-rollback on failed health checks.

You must provide: `server.js` with a `/health` endpoint, `package.json`.

Going off-road: you can replace any piece, but you own it —
the platform team supports only the paved road.
EOF
cat PLATFORM.md
```

**Expected output:** A short contract between platform team and developers.

**Why it matters:** The difference between a platform and a ticket queue is documentation and self-service. If developers can't discover the paved road, it doesn't exist.
