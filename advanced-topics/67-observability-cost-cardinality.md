# Observability Cost & Cardinality Interview Preparation Guide

*How to Answer Observability Cost & Cardinality Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Cardinality Basics](#cardinality-basics)
2. [Where the Bill Comes From](#where-the-bill-comes-from)
3. [Controlling Cost Without Going Blind](#controlling-cost-without-going-blind)

---

## Cardinality Basics

### Q1: What is cardinality, in plain terms?

**How to Answer:**

"I'd explain cardinality as the number of unique combinations a metric can take. A metric like http_requests_total with labels for method, status, and path is cheap if path has ten values — that's a few hundred series. It explodes the moment path becomes a user ID or a request ID, because then every request creates a brand-new series. The interview point is simple: labels are dimensions, and the total series count is the product of every label's values, not the sum. That's why one careless label can do more damage than a hundred new metrics."

**Key Point:** "Cardinality = unique label combinations, and it multiplies — one high-cardinality label like user ID can turn one metric into millions of series."

---

### Q2: Why does high cardinality hurt more than just the bill?

**How to Answer:**

"Cost is the symptom people notice first, but the real pain is operational. Every series has to be indexed, stored, and queried, so dashboards get slow, alerts evaluate late, and the monitoring system itself starts falling over during the incident you actually need it for. I've seen teams whose Prometheus slowed to a crawl exactly when traffic spiked, because the spike created new series faster than the old ones expired. So high cardinality isn't just expensive — it makes your observability unreliable at the worst possible moment."

**Key Point:** "High cardinality slows queries, delays alerts, and can take down monitoring during the incident — cost is only the first symptom."

---

### Q3: Which labels are safe, and which are dangerous?

**How to Answer:**

"Safe labels have a small, bounded set of values you control: method, status code, service name, environment, availability zone. Dangerous labels are unbounded or user-controlled: user ID, email, request ID, session ID, IP address, full URL path with IDs in it, and error messages with dynamic text. My rule is: if I can't list every possible value in advance, it doesn't go on a metric. That data belongs in logs or traces, where one extra field costs almost nothing, not on a metric where it creates a permanent series."

**Key Point:** "Bounded labels you control are safe; unbounded, user-controlled values belong in logs or traces, never on metrics."

---

### Q4: How do you spot a cardinality problem before it becomes a bill?

**How to Answer:**

"I look at three signals. First, active series count over time — a steady climb with no new services means a label is leaking values. Second, the top series by label, using the monitoring system's own cardinality analysis, which usually points straight at the offending label. Third, ingestion rate versus query rate: if you're ingesting far more than anyone queries, you're paying to store detail nobody uses. In an interview, I'd name the tool for the system in question — Prometheus's tsdb status, or the vendor's cardinality explorer — but the habit is the same: watch series count like you'd watch CPU."

```promql
# Series count per metric — the fastest way to find the offender
topk(10, count by (__name__)({__name__=~".+"}))
```

**Key Point:** "Watch active series count, rank metrics by series, and compare ingestion to actual queries — a climbing series count with no new services is a label leak."

---

## Where the Bill Comes From

### Q5: What actually drives an observability bill?

**How to Answer:**

"Four things, and metrics are only one of them. Ingested volume — how much data you send — is the biggest lever for logs and traces. Retention — how long you keep it — multiplies everything, because a month of data costs roughly thirty times a day of it. Cardinality multiplies metric cost, because each series is stored and indexed separately. And query or alert evaluation cost is the hidden one: heavy dashboards that everyone leaves open can cost as much as ingestion. When I get a surprise bill, I check in that order: volume, retention, cardinality, then query load."

**Key Point:** "Volume, retention, cardinality, and query load — in that order — are what actually drive the bill."

---

### Q6: Why are logs and traces priced so differently from metrics?

**How to Answer:**

"Because they're different shapes of data. A metric is a number plus labels, tiny and regular, so it's cheap per point but expensive when series explode. A log line or a span is a whole event with free-form text, so its cost scales with how many events you produce, not how many combinations they have. That's why the classic mistake is logging every request at INFO in production: you pay per event, and a traffic spike becomes a billing spike. The fix is sampling and levels — DEBUG off in production, INFO for important events only, and full detail only when you're tracing a specific problem."

**Key Point:** "Metrics cost per series, logs and traces cost per event — so log volume, not label count, is what makes a log bill spike."

---

### Q7: What is tail-based sampling, and when is it worth it?

**How to Answer:**

"Head-based sampling decides at the start — keep one in a hundred traces, drop the rest, and you never see the interesting ones you dropped. Tail-based sampling waits until the trace finishes, then keeps the ones that matter: errors, slow requests, and traces that hit a service you care about. It's worth it when errors are rare but expensive to miss, which is most production systems. The trade-off is real, though: you need a collector that buffers traces and makes the decision, which adds latency and infrastructure of its own. I'd use it for traces, never as a fix for a metrics cardinality problem — those are different problems."

**Key Point:** "Tail sampling keeps errors and slow traces after they finish, unlike head sampling's blind one-in-a-hundred — worth it for traces, useless for metric cardinality."

---

### Q8: How does retention policy become a cost decision?

**How to Answer:**

"I tier retention by how the data is actually used. Raw, high-resolution data is for live debugging, so I keep it for days, not months. Aggregated data — hourly rollups, downsampled series — is for trend questions, so it can live for months cheaply. And compliance or audit data is the only thing that justifies long raw retention, and even then it usually belongs in cheap object storage, not in the querying system. The mistake I see is one retention rule for everything, which means you're paying hot-storage prices for data nobody has queried in weeks."

**Key Point:** "Raw data for days, rollups for months, audit data in cheap storage — one retention rule for everything is how you pay hot prices for cold data."

---

## Controlling Cost Without Going Blind

### Q9: How do you cut cardinality without losing the ability to debug?

**How to Answer:**

"I never just delete a label and hope nobody notices — I move the detail somewhere cheaper. A high-cardinality value like user ID comes off the metric and goes into a log line or a trace attribute, linked by a trace ID or an exemplar, so I can still jump from a metric spike to the exact requests behind it. For paths, I normalize: /users/123 becomes /users/:id before it ever becomes a label. And I drop labels at ingestion with relabeling rules, so the expensive series are never created at all. The principle is: metrics answer 'how many and how fast,' logs and traces answer 'which ones' — don't make metrics do both jobs."

```yaml
# Drop the user_id label before it creates a series (Prometheus relabeling)
- action: labeldrop
  regex: user_id
```

**Key Point:** "Move detail to logs/traces via exemplars, normalize paths to route templates, and drop expensive labels at ingestion — metrics answer 'how many,' not 'which ones.'"

---

### Q10: What are recording rules and downsampling actually for?

**How to Answer:**

"They're both ways to pay once for a question you ask a hundred times. A recording rule precomputes an expensive query — say, error rate by service — into a new, cheap series, so every dashboard reads the precomputed answer instead of scanning raw data. Downsampling does the same thing over time: after a week, you don't need per-second resolution, so you keep one point per minute and throw the rest away. The trap is precomputing everything, which just recreates the cardinality problem in a new form. I only record queries that are actually slow or frequently used, and I measure before and after."

**Key Point:** "Recording rules precompute slow, frequent queries; downsampling trades old resolution for cheap storage — record only what dashboards actually use."

---

### Q11: How do you set up cost guardrails so this doesn't happen again?

**How to Answer:**

"I put cardinality and spend in the same place as any other production budget: an alert on series count growth, a dashboard showing cost per team or service, and a CI check that fails when a new metric introduces a high-cardinality label. The cultural part matters as much as the tooling — someone has to own the observability budget, and teams need to see their own share of it, otherwise nobody feels the cost of their labels. I also review the top ten most expensive metrics every month, because the offenders change as the product changes. Guardrails aren't about saying no to observability; they're about making its cost visible before the invoice does."

**Key Point:** "Alert on series growth, show cost per team, and check new labels in CI — make observability cost visible before the invoice does."

---

### Q12: Walk me through a real cardinality incident and how you'd fix it.

**How to Answer:**

"I'd tell it as a story with numbers. A service started emitting http_requests_total with a path label that included order IDs, and active series went from two hundred thousand to four million in a weekend. Dashboards slowed, then alerts started lagging, and the bill for the month doubled. The fix had three steps, in order: first, an emergency relabeling rule to drop the order ID label so no new series were created; second, a deploy that normalized paths to route templates, so the label was safe going forward; third, moving the per-order detail into traces, linked by exemplar, so debugging didn't lose anything. Then I'd add the guardrail — a series-count alert and a CI check on new labels — so the same label can't come back quietly. Interviewers remember incidents told with a number, a fix in order, and a guardrail at the end."

**Key Point:** "Stop the leak with relabeling, fix the source with normalized paths, move detail to traces, then add the alert and CI check so it can't return quietly."

---
