# Load Testing & Performance Engineering at Scale Interview Preparation Guide

*How to Answer Load Testing Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Load Testing Fundamentals](#load-testing-fundamentals)
2. [Finding & Fixing Bottlenecks](#finding--fixing-bottlenecks)
3. [Performance in Production](#performance-in-production)

---

## Load Testing Fundamentals

### Q1: Load testing, stress testing, soak testing — what is each one, and why do teams mix them up?

**How to Answer:**

"I'd draw the line by the question each test answers. Load testing asks: does the system meet its targets at the traffic we expect — say 5,000 requests per second with p95 under 300ms. Stress testing asks: where does it break — I keep adding load past the expected peak until something fails, so I know the breaking point instead of guessing it. Soak testing asks: does it survive at normal load for a long time — 12 hours at steady traffic, to catch memory leaks and connection pool exhaustion that only show up over time. Teams mix them up because they run one test and call it all three, which means they never learn the breaking point and never catch the slow leak. My line: load proves the target, stress finds the cliff, soak finds the leak — three tests, three different answers."

**Key Point:** "Load proves the target at expected traffic, stress finds the breaking point past it, soak finds the slow leak over hours — one test can't answer all three."

---

### Q2: How do you pick a load testing tool — k6 vs JMeter vs Locust vs Gatling?

**How to Answer:**

"I'd pick on three things: who writes the tests, what protocol I need, and how it runs in CI. k6 is my default — tests are plain JavaScript, it runs in CI with one command, and its thresholds fail the build when p95 goes red, so performance becomes a gate, not a report. JMeter is the old workhorse — it does everything including weird protocols, but the tests are XML in a GUI and they're painful to review in a pull request. Locust is Python, which wins when the team already thinks in Python and wants custom user behaviour that's hard to express in a script. Gatling is strong for high-throughput HTTP with a nice Scala DSL, but the learning curve is real. My line: k6 for a DevOps team that lives in CI, JMeter when the protocol is unusual, Locust when the team is Python-first."

**Key Point:** "k6 is the default — JavaScript tests that run in CI and fail the build on p95; JMeter for unusual protocols, Locust for Python teams, Gatling for high-throughput HTTP with a Scala DSL."

---

### Q3: How do you design a realistic load test — VUs, request rate, think time, test data?

**How to Answer:**

"I'd start from real traffic, not from a round number. I pull the production access logs for a peak hour and build the test from them: which endpoints get hit, in what ratio, and how many requests per second at the peak. Virtual users are just a way to generate that rate — I set the VUs and the pacing so the test produces the same requests per second as the real peak, and I add think time between requests so one VU behaves like a real user instead of a machine gun. Test data has to be realistic too: a thousand VUs hitting the same product ID tests the cache, not the database — I seed a pool of IDs and randomise. The mistake I call out is ramping straight to the target in one jump, because then I can't tell whether a failure is a real bottleneck or just the system warming up. My line: the test is a recording of real traffic, not a number I invented — same endpoints, same ratios, same rate, real data."

```javascript
// A k6 test that mirrors real traffic: realistic mix, think time, and a threshold that fails CI.
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '5m', target: 100 },   // ramp up slowly — no jump straight to peak
    { duration: '10m', target: 100 },  // hold at the expected peak
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: { http_req_duration: ['p(95)<300'], http_req_failed: ['rate<0.01'] },
};

const productIds = [101, 204, 317, 458, 592]; // a pool of real IDs — not one ID for everyone
export default function () {
  const id = productIds[Math.floor(Math.random() * productIds.length)];
  const res = http.get(`https://api.example.com/products/${id}`);
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1 + Math.random() * 2); // think time — a real user pauses between clicks
}
```

**Key Point:** "Design from production logs — same endpoints, same ratios, same request rate, a pool of real IDs — and ramp slowly, because jumping straight to peak hides whether a failure is real or just warm-up."

---

### Q4: Which metrics actually matter in a load test — and which ones mislead?

**How to Answer:**

"I'd watch four numbers and ignore the rest until those four are clean. Throughput — requests per second the system actually served, not the rate I asked for, because the gap between the two is the first sign of trouble. Latency percentiles — p95 and p99, never the average, because an average of 100ms can hide a p99 of 5 seconds that real users feel. Error rate — the percentage of failed requests, and which errors they are, because a 1% error rate that's all 500s is a different story from 1% that's all 429s. And saturation — CPU, memory, connection pools, and queue depth on the system under test, because a test that passes while the database is at 98% CPU hasn't proven anything. The metric that misleads most is the average latency — one slow request in a hundred barely moves the average and ruins the p99. My line: throughput, p95/p99, error rate, saturation — and never trust an average."

**Key Point:** "Throughput served, p95/p99 latency, error rate by type, and saturation on the system under test — and never trust an average, because it hides the slow 1% that users actually feel."

---

## Finding & Fixing Bottlenecks

### Q5: The test is red — how do you find the real bottleneck: app, database, network, or downstream?

**How to Answer:**

"I'd work the request's path and measure each hop instead of guessing. First I check the app servers — if CPU is pegged and latency grows with load, the app is the bottleneck, and I profile it before I scale it. Then the database — I check active sessions, slow queries, and lock waits, because a query that takes 20ms at 10 users takes 2 seconds at 1,000 if it's doing a full table scan. Then the network — I check connection counts and whether the load balancer or a proxy is queuing, because a saturated connection pool looks exactly like a slow database. Then downstream services — I check their latency and error rate, because my service is only as fast as the slowest call it makes. The trap is scaling the app when the database is the bottleneck — you just get more app servers waiting on the same database. My line: measure each hop in order, fix the slowest one, and re-run — one bottleneck at a time."

```bash
# The four checks that find the bottleneck — run them while the test is red:
# 1. App: is it CPU-bound or waiting?
top -b -n 1 | head -15; ss -s | grep -E "TCP|estab"
# 2. Database: what is it actually doing right now?
#    Postgres: SELECT pid, state, wait_event_type, query FROM pg_stat_activity WHERE state = 'active';
#    MySQL:    SHOW PROCESSLIST;  -- look for long-running queries, not just many connections
# 3. Network/pools: are requests queuing before they even reach the app?
curl -s http://localhost:9090/metrics | grep -E "http_server_requests|hikaricp_connections_active"
# 4. Downstream: is a dependency slow?
curl -w "%{time_total}\n" -o /dev/null -s https://payments.example.com/healthz
```

**Key Point:** "Measure each hop in order — app CPU, database active queries and locks, connection pools and queues, downstream latency — and never scale the app to fix a database bottleneck."

---

### Q6: What is saturation — and how do you spot it before users do?

**How to Answer:**

"I'd say saturation is when a resource is so busy that new work starts queuing instead of running, and latency explodes while throughput flatlines. The classic sign is the knee in the curve: as I add load, throughput climbs and latency stays flat, then suddenly latency shoots up and throughput stops growing — that's the saturation point, and it's where the system was already full. I spot it by watching utilisation against the queue: CPU at 70% with a growing run queue, a connection pool with zero idle connections, a message queue whose consumer lag keeps growing. The mistake is watching only CPU — a database can be saturated on locks or I/O while CPU sits at 40%, and a service can be saturated on threads while CPU is idle. My line: saturation is the knee where latency explodes and throughput flatlines — watch utilisation plus queue depth, not CPU alone."

**Key Point:** "Saturation is the knee where latency explodes and throughput flatlines — spot it by watching utilisation plus queue depth, because a database can be saturated on locks while CPU sits at 40%."

---

### Q7: How do you load test a database without destroying production data or the production database?

**How to Answer:**

"I'd say the rule is simple: never load test production, and never point a load test at a database you can't rebuild. The test runs against staging with a restored copy of production data — same volume, same distribution, anonymised — so the query plans and index behaviour are realistic. I seed the data first, because an empty database answers every query in 2ms and proves nothing. Writes go to a test schema or a disposable database, and I snapshot before the run so I can restore in minutes. And I cap the test's blast radius — separate credentials, a separate connection pool, and a kill switch — so a runaway test can't starve the staging database for everyone else. The mistake I call out is running the load test against a fresh empty database and declaring the system fast — it's fast because there's nothing in it. My line: restore real data to staging, seed it, snapshot it, and never let a test touch production."

**Key Point:** "Never load test production — restore anonymised production data to staging, seed it to real volume, snapshot before the run, and cap the blast radius with separate credentials and a kill switch."

---

### Q8: How do you handle third-party dependencies and rate limits under load?

**How to Answer:**

"I'd treat every third-party call as a bottleneck I don't control. First I find the rate limit — the payment gateway allows 100 requests per second, the SMS provider 10 — and I design the test to stay under it, or I get a higher limit for the test window. Then I mock the dependency in the load test when the point is testing my service, not their API — a stub that answers in 20ms with realistic responses. And I test the failure mode separately: what happens when the payment gateway times out — does my service fail fast with a circuit breaker, or does it hang every request for 30 seconds and exhaust its own thread pool? The mistake is load testing your service while hammering a real third-party API at 10x its limit — you get 429s, you think your service is broken, and you've burned the vendor relationship. My line: know their limit, mock them when testing yourself, and test the timeout path on purpose."

**Key Point:** "Know the vendor's rate limit, mock the dependency when testing your own service, and test the timeout path separately — a slow third party should trip a circuit breaker, not exhaust your thread pool."

---

## Performance in Production

### Q9: How do you run load tests safely — and why not just test against production?

**How to Answer:**

"I'd run them in staging, on infrastructure that matches production as closely as I can afford — same instance types, same database size class, same autoscaling rules — because a test on a tiny staging box tells me nothing about production. The load generators run outside the system, in their own network, so the generator itself isn't the bottleneck — I've seen tests where the generator ran out of CPU and everyone blamed the app. And I schedule tests when nobody else is using staging, with a clear owner and a stop condition: if error rate crosses 5%, the test stops automatically. Testing production directly is a different thing — it's called a game day, it needs real traffic shifted gradually, an incident commander, and a rollback plan, and it's never a load test. My line: staging that mirrors production, generators outside the system, an automatic stop condition — and production load tests are game days, not experiments."

**Key Point:** "Load test in staging that mirrors production, with generators outside the system and an automatic stop on high errors — testing production directly is a game day with an incident commander, not a load test."

---

### Q10: How do performance budgets and SLOs connect — and how do you enforce them in CI?

**How to Answer:**

"I'd say a performance budget is an SLO with teeth — it's the p95 target written down where the pipeline can check it. The SLO says the checkout API answers in under 300ms at p95; the budget says the CI pipeline fails the pull request if a load test against the preview environment breaks that number. I wire it in with thresholds in the load test itself — k6 fails the run when p95 crosses the budget, and the pipeline blocks the merge. The key is keeping the test small enough to run on every PR — a 5-minute smoke load test, not the full 2-hour soak — so developers get feedback in minutes, not next week. And I version the budget with the code, so when the SLO changes, the test changes in the same pull request. The mistake is a budget that lives in a wiki nobody reads — if the pipeline doesn't enforce it, it isn't a budget, it's a wish. My line: SLO in the contract, budget in the pipeline, small test on every PR, and the budget lives in the same repo as the code."

```yaml
# A performance budget that lives in CI — the pipeline fails the PR if p95 breaks it.
# .github/workflows/perf-budget.yml (excerpt)
- name: Run performance budget check
  run: |
    k6 run --out json=perf.json load-tests/checkout-smoke.js
    # k6 thresholds in the script already fail the run when p95 > 300ms;
    # the workflow just needs the exit code — no extra parsing.
- name: Upload results
  uses: actions/upload-artifact@v4
  with:
    name: perf-results
    path: perf.json
```

**Key Point:** "A performance budget is an SLO the pipeline enforces — k6 thresholds fail the PR when p95 crosses the target, the test is small enough to run on every PR, and the budget lives in the same repo as the code."

---

### Q11: The load test passes but production still falls over — what went wrong?

**How to Answer:**

"I'd look for the four ways a passing test lies. One: the test data was too clean — one product ID, one user, everything cached — so the database never worked. Two: the test skipped the expensive path — it hit the fast GET endpoint and never ran the checkout flow that does five database writes. Three: staging didn't match production — a smaller database, no autoscaling, or a load balancer rule that production has and staging doesn't. Four: the failure was in something the test never touched — a cron job that runs at midnight, a third-party webhook that fires under real traffic, or a cache that expires under real load patterns. The fix is always the same: compare the test's traffic shape against the production access logs, find the gap, and close it. My line: a passing test that doesn't look like production is a rehearsal for a play nobody performs — match the traffic shape, then trust the result."

**Key Point:** "A passing test lies when the data is too clean, the expensive path is skipped, staging doesn't match production, or the failure lives in something the test never touched — compare the test against production access logs and close the gap."

---

### Q12: Walk me through a load-test plan for a checkout service before a big sale.

**How to Answer:**

"I'd take a checkout service expecting 10x normal traffic for a 6-hour sale, because every decision gets concrete. First I set the target from last year's sale plus growth — say 8,000 checkouts per minute with p95 under 500ms and error rate under 0.1% — and I write it down as the pass criteria before I run anything. Then I build the test from production logs: the real mix of browse, add-to-cart, and checkout, with a pool of product IDs and test accounts, and the payment gateway mocked except for one dedicated run against its sandbox. I ramp in stages — 25%, 50%, 100% of the target, holding at each stage — and I watch throughput, p95, error rate, and saturation on the app, the database, and the queue together. I find the bottleneck, fix it, and re-run until the system holds 150% of the target — the headroom is the point, because the sale will exceed the forecast. And I finish with a soak: 2 hours at the target to catch the slow leak. My interview close: target from last year plus growth, test from real logs, ramp in stages, fix one bottleneck at a time, prove 150% headroom, then soak — and the runbook for the sale itself is written from the numbers the test produced."

**Key Point:** "Target from last year's sale plus growth, test built from real logs with the payment gateway mocked, ramp in stages, fix one bottleneck at a time, prove 150% headroom, then soak — and write the sale-day runbook from the test's numbers."
