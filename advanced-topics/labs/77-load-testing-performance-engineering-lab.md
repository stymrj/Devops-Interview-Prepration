# Load Testing & Performance Engineering at Scale — Hands-On Lab

Nine drills that turn load testing answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Write your first k6 test and find the p95, not the average

**Goal:** A working k6 test against a local app, with p95 and p99 reported — and proof the average hides the slow requests.

**Steps:**

```bash
# Install k6, then run a tiny app locally (any HTTP service — nginx is enough):
docker run -d --name web -p 8080:80 nginx
# Write a k6 test that hits it with 50 VUs for 1 minute:
cat > first-test.js <<'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
export const options = {
  vus: 50,
  duration: '1m',
  thresholds: { http_req_duration: ['p(95)<200'] },
};
export default function () {
  const res = http.get('http://localhost:8080/');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
EOF
k6 run first-test.js
```

**Expected output:** The k6 summary showing avg, p90, p95, p99 side by side — and one line noting how far apart the average and the p99 are, even on a healthy local service.

**Why it matters:** This is Q4 made concrete — the average hides the slow 1%, and the p95 is the number interviewers expect you to quote.

---

## Exercise 2: Load vs stress vs soak — run all three and tell them apart

**Goal:** Three runs against the same app, each answering a different question, with the breaking point and the leak named.

**Steps:**

```bash
# Load: hold at the expected peak (say 100 VUs) for 10 minutes and record p95 + error rate.
# Stress: ramp past the peak — 100, 200, 400, 800 VUs — until errors climb or p95 explodes.
# Soak: hold at 50 VUs for 2 hours (or 30 minutes for a first run) and watch memory over time.
# k6 stages make the ramp explicit:
cat > stages.js <<'EOF'
import http from 'k6/http';
export const options = {
  stages: [
    { duration: '2m', target: 100 },  // load
    { duration: '2m', target: 400 },  // stress — past the peak
    { duration: '2m', target: 50 },   // back down
  ],
};
export default function () { http.get('http://localhost:8080/'); }
EOF
k6 run stages.js
```

**Expected output:** Three numbers — the p95 at expected load, the VU count where p95 explodes (the breaking point), and whether memory grew during the soak — each labelled with the test that produced it.

**Why it matters:** This is Q1 in one drill — load proves the target, stress finds the cliff, soak finds the leak, and one test can't do all three.

---

## Exercise 3: Build a realistic test from production logs

**Goal:** A load test whose endpoint mix matches real traffic, built from a log file — not from a round number.

**Steps:**

```bash
# Take an access log (or make a small sample) and count the endpoint mix:
awk '{print $7}' access.log | cut -d'?' -f1 | sort | uniq -c | sort -rn | head -10
# Turn the top 5 endpoints into a k6 test with the same ratios — e.g. 60% GET /products,
# 25% GET /products/:id, 10% POST /cart, 5% POST /checkout — using a pool of real IDs
# and think time between requests (see Q3's script for the shape).
# Run it and compare the requests-per-second it generates with the log's peak hour.
```

**Expected output:** The endpoint mix from the log, the matching test script, and the requests-per-second the test produced versus the log's real peak — with the gap (if any) written down.

**Why it matters:** This is Q3 made real — a test built from logs is a recording of real traffic, and a test built from a round number is a guess.

---

## Exercise 4: Find the bottleneck — app, database, or downstream?

**Goal:** A red test with the bottleneck named and proven by a measurement, not a guess.

**Steps:**

```bash
# Run a load test against an app backed by a database (a local Docker Compose stack is enough).
# While it runs, check each hop in order (Q5's commands):
# 1. App CPU: top / docker stats — is it pegged?
# 2. Database: pg_stat_activity or SHOW PROCESSLIST — what's actually running?
# 3. Pools/queues: connection pool active vs idle, queue depth.
# 4. Downstream: curl a dependency's health endpoint and time it.
# Pick the slowest hop, fix one thing (add an index, raise the pool, cache one call),
# and re-run to prove the p95 moved.
```

**Expected output:** The hop you blamed, the measurement that proved it, the one fix you made, and the p95 before and after.

**Why it matters:** This is Q5 as a skill — measure each hop in order, fix the slowest one, re-run — and never scale the app to fix a database bottleneck.

---

## Exercise 5: Catch saturation — find the knee in the curve

**Goal:** A graph (even hand-drawn from numbers) showing throughput flatlining while latency explodes — the saturation knee.

**Steps:**

```bash
# Ramp VUs in steps — 10, 25, 50, 100, 200, 400 — holding 1 minute at each step,
# and record throughput + p95 at every step (k6's summary gives you both).
# Plot the two series: throughput climbs, then stops; p95 stays flat, then shoots up.
# The step where that happens is the saturation point — note which resource was
# saturated at that step (CPU, pool, queue depth) from your monitoring.
```

**Expected output:** A small table — VUs, throughput, p95 — with the knee marked, and the saturated resource named.

**Why it matters:** This is Q6 made visible — saturation is the knee where latency explodes and throughput flatlines, and you only see it by ramping in steps.

---

## Exercise 6: Load test a database safely — seed it, snapshot it, restore it

**Goal:** A load test against a seeded database that you can restore in minutes — with the empty-database trap demonstrated.

**Steps:**

```bash
# Start Postgres in Docker, seed it with realistic volume (e.g. 100k rows across two tables):
docker run -d --name pg -e POSTGRES_PASSWORD=test -p 5432:5432 postgres:16
# Seed: pgbench -i -s 10 pgbench  (or your own seed script with realistic row counts)
# Snapshot before the test:
docker exec pg pg_dump -U postgres pgbench > before.sql
# Run the same query load twice — once against the seeded database, once against an empty one —
# and compare p95. Then restore from the snapshot to prove the rollback works:
docker exec -i pg psql -U postgres pgbench < before.sql
```

**Expected output:** The p95 on the seeded database vs the empty one, and the time the restore took — plus one line on why the empty-database number is a lie.

**Why it matters:** This is Q7 in one drill — an empty database answers in 2ms and proves nothing, and a test you can't restore is a test you can't rerun.

---

## Exercise 7: Mock a third party and test the timeout path

**Goal:** A test that proves your service fails fast when a dependency times out — instead of hanging every request.

**Steps:**

```bash
# Stand up a stub for a slow dependency (a tiny server that sleeps 10s before answering),
# and point your app at it. Then:
# 1. Load test with the stub answering fast (20ms) — record p95.
# 2. Make the stub sleep 10s and load test again — watch what happens to your app's
#    p95, thread pool, and error rate. Does it fail fast, or does it hang?
# 3. Add a timeout + circuit breaker around the call (e.g. 500ms timeout, breaker opens
#    after 5 failures) and re-run step 2 — the p95 should recover and errors should
#    become fast 503s instead of 30s hangs.
```

**Expected output:** The p95 and error shape with a fast dependency, a slow dependency without a breaker, and a slow dependency with a breaker — three rows, one lesson.

**Why it matters:** This is Q8 made concrete — a slow third party should trip a circuit breaker, not exhaust your thread pool, and you only learn that by testing the timeout path on purpose.

---

## Exercise 8: Put a performance budget in CI

**Goal:** A pull request that fails when p95 breaks the budget — the budget enforced by the pipeline, not a wiki.

**Steps:**

```bash
# Take the k6 script from Exercise 1, add a threshold (p95 < 200ms, error rate < 1%),
# and wire it into a GitHub Actions workflow (see Q10's YAML for the shape).
# Push a change that deliberately slows the endpoint (e.g. sleep 500ms in the handler)
# and watch the workflow fail. Then revert it and watch it go green.
# Keep the test short — a 2-minute smoke test — so it can run on every PR.
```

**Expected output:** A failing workflow run with the threshold breach in the log, a green run after the revert, and the runtime of the smoke test.

**Why it matters:** This is Q10 in one drill — a budget the pipeline doesn't enforce is a wish, and a 2-minute smoke test on every PR is what makes it real.

---

## Exercise 9: Write the sale-day load-test plan — and prove 150% headroom

**Goal:** A one-page written plan for a checkout service at 10x traffic, with the pass criteria, the ramp, and the headroom number.

**Steps:**

```bash
# Pick a target from a real baseline (your Exercise 3 numbers, or a stated assumption):
# e.g. 8,000 checkouts/minute, p95 < 500ms, error rate < 0.1%.
# Write the plan: the endpoint mix from logs, the ramp stages (25% → 50% → 100% → 150%),
# the mocked dependencies, what you watch at each stage (throughput, p95, errors, saturation),
# and the stop condition. Then run at least the 100% stage against your local stack
# and record whether it held — and what broke first if it didn't.
```

**Expected output:** The one-page plan with the pass criteria, the ramp, and the measured result at 100% — plus the bottleneck you'd fix first to reach 150%.

**Why it matters:** This is Q12 as a deliverable — a plan with a target, a ramp, and a headroom number is what an interviewer asks for, and running even one stage of it is what makes the answer believable.
