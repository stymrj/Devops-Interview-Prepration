# Observability Cost & Cardinality — Hands-On Lab

Nine drills that turn cardinality answers into real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Count your series and find the biggest metric

**Goal:** A ranked list of metrics by series count, with one clear offender.

**Commands:**

```promql
# In Prometheus: series per metric, top 10
topk(10, count by (__name__)({__name__=~".+"}))
```

**Expected output:** One or two metrics holding a disproportionate share of all series.

**Why it matters:** This is the Q4 starting point — you can't fix cardinality until you can rank it.

---

## Exercise 2: Prove a label explodes cardinality

**Goal:** See the multiplication effect with your own numbers, not just theory.

**Commands:**

```bash
# Count series with and without a suspect label, e.g. path vs path without IDs
# In PromQL, compare:
count(http_requests_total)
count by (method, status) (http_requests_total)
```

**Expected output:** The ungrouped count is far larger than the grouped one when a high-cardinality label is present.

**Why it matters:** This is Q1 and Q3 made concrete — the product of label values, not the sum.

---

## Exercise 3: Normalize paths before they become labels

**Goal:** A metric where /users/123 and /users/456 are one series, not two.

**Commands:**

```bash
# In your app or exporter, map concrete paths to route templates, e.g.:
# /users/123 -> /users/:id, /orders/abc-9 -> /orders/:id
# Then verify: count by (path) (http_requests_total) stays flat as IDs vary
```

**Expected output:** Series count stays flat while request IDs keep changing.

**Why it matters:** This is the Q9 source fix — normalization beats dropping data after the fact.

---

## Exercise 4: Drop an expensive label at ingestion

**Goal:** A relabeling rule that stops new series from being created.

**Commands:**

```yaml
# scrape_configs -> relabel_configs, applied before storage
- action: labeldrop
  regex: user_id
```

**Expected output:** After reload, no new series carry user_id, and series growth stops.

**Why it matters:** This is the Q9 emergency fix — drop at ingestion, not after storage.

---

## Exercise 5: Link a metric spike to traces with exemplars

**Goal:** From a metric, jump to the exact requests behind it, without a user_id label.

**Commands:**

```bash
# Enable exemplars in Prometheus and check a series carries exemplar data
# Query a histogram metric and inspect its exemplars for trace IDs
```

**Expected output:** A spike in the metric leads to trace IDs you can open directly.

**Why it matters:** This is the Q9 promise — you lose nothing by removing the label, because detail lives in traces.

---

## Exercise 6: Compare head-based and tail-based sampling

**Goal:** A sampling decision you can defend with a concrete trade-off.

**Commands:**

```bash
# Configure a collector with tail sampling: keep errors + p99-slow traces, drop the rest
# Send a mix of successful and failing requests, then count kept traces by outcome
```

**Expected output:** Nearly all error traces kept, most successful traces dropped, total volume far lower.

**Why it matters:** This is Q7 — tail sampling keeps what matters, head sampling drops it blindly.

---

## Exercise 7: Design a tiered retention policy

**Goal:** A retention plan with a number attached to each tier.

**Commands:**

```bash
# Write it down, then check it against real usage:
# Raw metrics/logs: 7 days (live debugging)
# Hourly rollups: 90 days (trends)
# Audit/compliance archive: object storage, 1 year+
# Verify: when was each tier last queried?
```

**Expected output:** A policy where no tier is kept longer than its last real query justifies.

**Why it matters:** This is Q8 — retention is a cost decision, tiered by actual use.

---

## Exercise 8: Precompute one expensive dashboard query

**Goal:** A recording rule that makes a slow dashboard fast.

**Commands:**

```yaml
# rules: precompute error rate by service once, read it everywhere
- record: service:error_rate_5m
  expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
        / sum by (service) (rate(http_requests_total[5m]))
```

**Expected output:** Dashboards reading service:error_rate_5m load noticeably faster than the raw query.

**Why it matters:** This is Q10 — pay once for a question asked a hundred times.

---

## Exercise 9: Set a series-count alert and a cost review habit

**Goal:** A guardrail that fires before the invoice does.

**Commands:**

```promql
# Alert when total series grows fast, e.g. 20% in an hour
# (adapt the threshold to your system)
increase(prometheus_tsdb_head_series[1h]) > 0
# Monthly habit: list your top 10 metrics by series and by cost, and review them as a team
```

**Expected output:** An alert that would have caught the Q12 incident on day one, plus a monthly review list.

**Why it matters:** This is Q11 — guardrails make cost visible before the invoice does.

---
