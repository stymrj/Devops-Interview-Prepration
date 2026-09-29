# SLI, SLO, SLA and Error Budgets — Hands-On Lab

*Turn SLO theory into numbers you can defend in an interview. You'll simulate a service, compute real SLIs, spend an error budget, and build burn-rate alerts. Work in `/tmp/slo-lab`.*

**Setup:**

```bash
mkdir -p /tmp/slo-lab && cd /tmp/slo-lab
# generate a month of fake request logs: ts,status,latency_ms
python3 - <<'EOF'
import random, time
random.seed(12)
now = int(time.time())
with open("requests.log", "w") as f:
    for i in range(43200):  # ~30 days, 1 req/min
        ts = now - (43200 - i) * 60
        r = random.random()
        # an "incident" on day 20-21: elevated errors + latency
        incident = 28800 <= i < 30240
        err = 0.05 if incident else 0.0008
        lat = random.gauss(900 if incident else 120, 60)
        status = 500 if random.random() < err else 200
        f.write(f"{ts},{status},{max(1,int(lat))}\n")
print("log ready")
EOF
wc -l requests.log
```

---

## Exercise 1 — Compute an availability SLI

**Goal:** Turn raw logs into the availability SLI: good events / total events.

**Commands:**

```bash
awk -F, '$2==200{c++} {t++} END{printf "SLI availability = %.4f%%\n", 100*c/t}' requests.log
```

**Expected output:** Something like `SLI availability = 99.93%` — the two-day incident drags it below four nines.

**Why it matters:** Every SLO conversation starts here. If you can't compute the SLI from raw data, the rest is hand-waving.

---

## Exercise 2 — Compute a latency SLI (p99)

**Goal:** Add the second classic SLI: share of requests under a latency threshold.

**Commands:**

```bash
awk -F, '{print $3}' requests.log | sort -n | awk '{a[NR]=$1} END{print "p50="a[int(NR*0.5)]"ms p99="a[int(NR*0.99)]"ms"}'
awk -F, '$3<=300{g++} {t++} END{printf "latency SLI (<300ms) = %.4f%%\n", 100*g/t}' requests.log
```

**Expected output:** p50 ≈ 120ms, p99 ≈ 900+ms (the incident inflates it); latency SLI maybe ~97% — fails a 99% target.

**Why it matters:** Averages lie; p99 tells you what your worst real users feel. Interviewers expect you to reach for percentiles, not means.

---

## Exercise 3 — Set the SLO and size the error budget

**Goal:** Convert "99.9% over 30 days" into minutes you can actually spend.

**Commands:**

```bash
python3 - <<'EOF'
slo = 0.999
window_min = 30 * 24 * 60
budget_min = (1 - slo) * window_min
print(f"SLO 99.9% over 30d  -> error budget: {budget_min:.1f} minutes")
print(f"SLO 99.95% over 30d -> error budget: {(1-0.9995)*window_min:.1f} minutes")
print(f"SLO 99.99% over 30d -> error budget: {(1-0.9999)*window_min:.1f} minutes")
EOF
```

**Expected output:** 43.2 min, 21.6 min, 4.3 min. One extra nine = 10x less room.

**Why it matters:** "How many minutes of downtime does 99.9% allow?" is a classic whiteboard question. 43.2 minutes/month is the number to memorize.

---

## Exercise 4 — Measure budget burned by your incident

**Goal:** Find how much of the 43.2-minute budget the day 20-21 incident consumed.

**Commands:**

```bash
python3 - <<'EOF'
errs = bad_min = 0
with open("requests.log") as f:
    for line in f:
        ts, status, lat = line.split(",")
        if status.strip() == "500":
            errs += 1
# ~1 req/min, so each error ≈ 1 bad minute of 100% failure-equivalent
print(f"errors: {errs} -> budget burned ≈ {errs:.0f} min of 43.2 min ({100*errs/43.2:.0f}%)")
EOF
```

**Expected output:** Roughly 70+ bad minutes — over 100% of the budget. The month's budget is gone.

**Why it matters:** This is the moment error budgets stop being abstract. One bad deploy day can spend the whole month — now "freeze features" is a number, not a vibe.

---

## Exercise 5 — Compute burn rate during the incident

**Goal:** Express the incident as burn rate: how many times faster than sustainable.

**Commands:**

```bash
python3 - <<'EOF'
# sustainable burn: 0.1% errors over 30d. During incident: ~5% errors.
sustainable = 0.001
incident_err = 0.05
print(f"burn rate during incident: {incident_err/sustainable:.0f}x")
print("page threshold (14x) crossed?" , incident_err/sustainable > 14)
EOF
```

**Expected output:** ~50x burn — well past the 14x fast-burn page threshold.

**Why it matters:** Burn rate is how the 30-day math becomes a page in the next 5 minutes. 14x fast-burn / 6x slow-burn are the numbers to know.

---

## Exercise 6 — Spot the bad SLI

**Goal:** Prove that a plausible-looking SLI can hide real user pain.

**Commands:**

```bash
# "server uptime" SLI: was the process alive? (always yes in our log)
echo "uptime SLI: 100.00% -- process never died"
# vs user-facing availability from Exercise 1
awk -F, '$2==200{c++} {t++} END{printf "user-facing SLI: %.4f%%\n", 100*c/t}' requests.log
```

**Expected output:** Uptime says 100%, users saw ~99.93% with a two-day incident. The SLI was green; users were not.

**Why it matters:** "Our SLO is green but users complain" is an interview favorite. The answer is always: you're measuring the wrong thing.

---

## Exercise 7 — Draft the SLO policy doc

**Goal:** Write the one-page contract that makes error budgets enforceable.

**Commands:**

```bash
cat > slo-policy.md <<'EOF'
# Checkout API — SLO policy
- SLI: % of requests returning 2xx within 300ms, 30-day rolling window
- SLO: 99.9% (error budget: 43.2 min/month)
- SLA (customer contract): 99.5% — credits below, exclusions: scheduled maintenance
- Budget > 50% remaining: ship normally
- Budget 10-50%: reliability work prioritized in sprint planning
- Budget < 10%: feature freeze, deploys gated by SRE review
- Fast burn (>14x, 1h+5m windows): page on-call
- Slow burn (>6x, 6h+30m windows): ticket, investigate within 2 days
- SLO reviewed quarterly with product; SLI changed if users complain while green
EOF
cat slo-policy.md
```

**Expected output:** The policy file prints. Nothing fancy — the value is the pre-agreed rules.

**Why it matters:** Error budgets only work as a contract signed before the incident. This doc is what you show product when the dashboard goes red.

---

## Exercise 8 — Present it like an interview answer

**Goal:** Summarize the month in 60 seconds, out loud.

**Commands:**

```bash
cat <<'EOF'
"Our checkout API ran 99.93% availability against a 99.9% SLO this month,
so we technically met it — but the latency SLI missed: 97% under 300ms
against a 99% target, driven by a two-day incident on the 20th that burned
the entire 43-minute error budget at ~50x burn rate. Action: the incident
root cause gets fixed this sprint, and we're adding a p99 latency SLI
because the old one averaged away the pain users actually felt."
EOF
```

**Expected output:** A spoken summary you can practice until it's natural.

**Why it matters:** Interviews don't ask you to recite definitions — they ask what happened and what you did. Numbers plus narrative is the whole game.
