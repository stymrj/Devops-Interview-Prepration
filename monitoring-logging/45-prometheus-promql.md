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
