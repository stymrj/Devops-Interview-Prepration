# Prometheus & PromQL Interview Preparation Guide

*How to Answer Prometheus & PromQL Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Prometheus Core Concepts](#prometheus-core-concepts)
2. [PromQL Query Language](#promql-query-language)
3. [Scraping Rules and Alerting](#scraping-rules-and-alerting)

---

## Prometheus Core Concepts

### Q1: What is Prometheus, and why does it pull instead of push?

**How to Answer:**

"I describe it as a pull-based time-series monitoring system built for dynamic environments. Instead of agents pushing data, Prometheus scrapes an HTTP /metrics endpoint on a schedule — which means I can tell from the `up` metric whether a target is healthy or the scrape itself failed. Pull keeps the failure modes honest: no data, no guesswork about whether the app died or the pipeline did. It's the backbone of most Kubernetes monitoring stacks, usually paired with Grafana."

**Key Point:** "Prometheus scrapes targets over HTTP on a schedule; the `up` metric tells you instantly if a target is down."

---

### Q2: What are the four metric types, and when do you use each?

**How to Answer:**

"I use counters for things that only go up — total HTTP requests, errors served, jobs completed. Gauges for things that go up and down — CPU usage, queue depth, active connections. Histograms when I need latency percentiles — they bucket observations so I can compute p95/p99 with `histogram_quantile`. Summaries also give quantiles but they're computed client-side, so I avoid them — I can't aggregate summaries across instances, and that kills them in a scaled-out service."

**Key Point:** "Counters go up, gauges go both ways, histograms give aggregatable percentiles, summaries don't aggregate — prefer histograms."

---

### Q3: How does Prometheus store data, and what does retention actually do?

**How to Answer:**

"Prometheus uses its own local TSDB — samples are grouped into chunks per time series, compressed, and written to disk in 2-hour blocks. By default it keeps 15 days of data, controlled by `--storage.tsdb.retention.time` and a size cap with `--storage.tsdb.retention.size`. The trap answer is treating it like a long-term store — it's not. For long retention I bolt on Thanos or Mimir for remote write, and keep Prometheus itself lean and fast."

**Key Point:** "Local TSDB in 2-hour blocks, 15-day default retention — Prometheus is short-term; Thanos/Mimir handles long-term storage."

---

### Q4: What is cardinality, and why does it take down Prometheus servers?

**How to Answer:**

"Cardinality is the number of unique time series — every unique label combination creates a new series in memory. The classic footgun is putting a user ID, request path with IDs, or pod hash into a label — suddenly millions of series and the TSDB eats all RAM. I've seen a single bad label take a Prometheus pod OOM. My rule: labels describe the dimension I'd group by in an alert — service, method, status code — never unbounded values. If I need that detail, it goes in logs or traces, not metrics."

**Key Point:** "Every unique label combo is a series in RAM — unbounded label values (user IDs, request IDs) will OOM your Prometheus."

---

## PromQL Query Language

### Q5: Instant vectors vs range vectors — what's the difference?

**How to Answer:**

"An instant vector is a set of values at one point in time — `http_requests_total` gives me the latest counter value per series right now. A range vector adds a time window — `http_requests_total[5m]` gives me all samples from the last 5 minutes. Range vectors only work with functions like `rate()` or `increase()` that need history. The interview trap: putting a range selector on a gauge query directly fails — PromQL expects an instant vector there unless a function consumes the range."

```promql
# Error rate over the last 5 minutes
rate(http_requests_total{status=~"5.."}[5m])
```

**Key Point:** "Instant vector = values right now; range vector = values over a window, and only range functions can consume it."

---

### Q6: rate() vs irate() — which one do you actually use?

**How to Answer:**

"I default to `rate()` — it averages the per-second increase over the window, which smooths out scrape hiccups. `irate()` only looks at the last two samples, so it's jumpy and I use it for fast-moving counters on graphs where I want responsiveness, never in alerts. The real trap: the window must be at least 2-4x the scrape interval — `rate(x[1m])` on a 30s scrape interval misses samples and gives wrong numbers. I use `[5m]` as my floor."

**Key Point:** "Use rate() in alerts and SLOs, irate() only for twitchy graphs — and keep the window 2-4x the scrape interval."

---

### Q7: How do by, without, and topk work in aggregations?

**How to Answer:**

"`by` keeps only the labels I list — `sum by (service) (rate(http_requests_total[5m]))` gives me total RPS per service. `without` drops the listed labels and keeps the rest. `topk` returns the K highest series — `topk(5, ...)` for my top 5 CPU-hungry pods. The trap: aggregating without `by` collapses everything into one number, which is what you want for a global SLO but terrible for finding which service is broken. I always aggregate at the level I'll page on."

```promql
# Top 5 pods by CPU usage
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
```

**Key Point:** "by keeps listed labels, without drops them, topk ranks — aggregate at the level you'd actually page on."

---

### Q8: How do you compute p95 latency in PromQL?

**How to Answer:**

"I use `histogram_quantile` on the histogram buckets — `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`. The `le` label must survive aggregation, so I sum `by (le)` — dropping it is the classic mistake that returns garbage. And it has to be the `_bucket` series, not `_sum` or `_count`. One caveat I mention: histogram_quantile interpolates between buckets, so bucket boundaries I choose at instrumentation time decide how accurate my p95 is."

**Key Point:** "histogram_quantile(0.95, sum by (le) (rate(..._bucket[5m]))) — keep the le label and query the _bucket series."

---

## Scraping Rules and Alerting

### Q9: How do service discovery and scrape configs work in practice?

**How to Answer:**

"In Kubernetes I use `kubernetes_sd_configs` so Prometheus discovers pods and services automatically as they come and go — no static target lists to maintain. I annotate services or use PodMonitors/ServiceMonitors from the Prometheus Operator, and relabeling decides which endpoints actually get scraped and what labels they carry. The scrape config sets the interval, timeout, and path. The interview trap: a target showing DOWN usually isn't the app — it's the scrape config, wrong port annotation, or network policy blocking Prometheus."

```yaml
scrape_configs:
  - job_name: 'my-app'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
```

**Key Point:** "Service discovery finds targets dynamically; relabeling filters them — a DOWN target is usually config, not the app."

---

### Q10: What is relabeling, and when do you drop or rewrite labels?

**How to Answer:**

"Relabeling runs before the scrape and rewrites the target's label set — keep, drop, replace, or hashmod actions. I use it to drop noisy targets, strip high-cardinality labels before they hit the TSDB, and map Kubernetes metadata like namespace and pod name onto clean labels. It also runs after scraping as metric_relabel_configs — that's my last line of defense against cardinality bombs, dropping series I don't need. If an interviewer asks for one use case: dropping the `pod` label on aggregated metrics I'll never alert per-pod on."

**Key Point:** "Relabeling rewrites labels pre-scrape; metric_relabel_configs drops series post-scrape — both fight cardinality."

---

### Q11: Recording rules vs alerting rules — how do you use each?

**How to Answer:**

"Recording rules precompute expensive queries into new series — `job:http_requests:rate5m` — so my dashboards and alerts query one series instead of re-running heavy aggregations every evaluation. Alerting rules evaluate an expression and fire to Alertmanager when it's true for the `for` duration. The `for` clause is what separates a blip from a real incident — I set it based on how fast the symptom becomes user-facing. Alertmanager then handles dedup, grouping, silences, and routing to PagerDuty or Slack."

```yaml
groups:
  - name: api-alerts
    rules:
      - alert: HighErrorRate
        expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
              / sum by (service) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations: { summary: "Error rate above 5% for {{ $labels.service }}" }
```

**Key Point:** "Recording rules precompute expensive queries; alerting rules fire to Alertmanager — the for clause turns blips into real pages."

---

### Q12: When do you use the Pushgateway, and what's the trap?

**How to Answer:**

"Only for short-lived jobs — batch scripts, cron jobs, CI pipelines — things that die before Prometheus can scrape them. The job pushes its final metrics to the Pushgateway, which holds them for scraping. The trap: using it for long-lived services. Stale metrics linger after the job dies, and a dead instance keeps reporting its last values as if it's alive — I've seen that mask real outages. For anything that lives longer than a scrape interval, expose /metrics and let Prometheus pull."

**Key Point:** "Pushgateway is only for short-lived jobs — for long-lived services it leaves stale metrics that mask outages."
