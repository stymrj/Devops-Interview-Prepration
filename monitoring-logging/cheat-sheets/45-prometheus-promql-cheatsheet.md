# Prometheus & PromQL — Cheat Sheet

## Prometheus in 30 seconds

- **Pull-based** TSDB: scrapes `http://target:port/metrics` on a schedule. `up` metric = 1 means the scrape worked.
- **Local TSDB**, 2-hour blocks, 15-day default retention (`--storage.tsdb.retention.time`, `--storage.tsdb.retention.size`).
- Long-term storage: **Thanos / Mimir** via remote write. Prometheus itself stays short-term.
- **Cardinality** = unique time series = unique label combos. Unbounded label values (user IDs, request IDs) → OOM.

## Metric types

| Type | Direction | Use for |
|---|---|---|
| Counter | up only | requests total, errors total |
| Gauge | up/down | CPU %, queue depth, connections |
| Histogram | buckets | latency percentiles (aggregatable) |
| Summary | client-side quantiles | avoid — can't aggregate across instances |

## Scrape config essentials

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['app:8080']
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        regex: '.*:badport'
        action: drop
```

- **Service discovery**: `kubernetes_sd_configs` (role: endpoints/pods), or Operator's ServiceMonitor/PodMonitor.
- **Relabeling**: rewrite labels pre-scrape; `metric_relabel_configs` drops series post-scrape (cardinality guard).
- **Pushgateway**: only for short-lived batch/CI jobs. Never for long-lived services (stale metrics mask outages).

## PromQL selectors

```promql
http_requests_total                          # instant vector
http_requests_total[5m]                       # range vector
http_requests_total{job="api",status=~"5.."} # label matchers (=, !=, =~, !~)
```

## Functions you'll actually use

| Function | What it does |
|---|---|
| `rate(v[5m])` | per-second avg increase — alerts, SLOs, dashboards |
| `irate(v[5m])` | last-two-samples rate — jumpy, graphs only |
| `increase(v[1h])` | total increase over window |
| `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))` | p95 latency |
| `predict_linear(v[1h], 3600)` | forecast 1h ahead (disk-full alerts) |
| `delta()`, `deriv()` | gauge change over window |

Window rule: **≥ 2–4× scrape interval** (use `[5m]` minimum on 15s scrapes).

## Aggregation

```promql
sum by (service) (rate(http_requests_total[5m]))   # keep listed labels
sum without (pod) (rate(http_requests_total[5m])) # drop listed labels
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
count(up == 0)                                     # how many targets down
```

## Rules

```yaml
groups:
  - name: api
    interval: 30s
    rules:
      - record: job:http_requests:rate5m          # recording rule
        expr: sum by (job) (rate(http_requests_total[5m]))
      - alert: HighErrorRate                      # alerting rule
        expr: sum by (s) (rate(http_requests_total{status=~"5.."}[5m]))
              / sum by (s) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels: { severity: page }
        annotations: { summary: "Error rate high on {{ $labels.s }}" }
```

- **Recording rules**: precompute expensive queries → cheap series for dashboards/alerts.
- **`for:`** turns blips into pages. **Alertmanager**: dedup, grouping, silences, routing (Slack/PagerDuty).

## Copy-paste queries

```promql
up == 0                                            # dead targets
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))  # CPU utilization
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes  # free mem ratio
predict_linear(node_filesystem_free_bytes[1h], 4*3600) < 0   # disk full in 4h
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))  # error rate
```

## Debugging a missing metric

1. **Status → Targets**: UP or DOWN? Read the scrape error.
2. Wrong port/path/annotation? Check relabeling in **Status → Service discovery**.
3. Query the raw series — did the label set change?
4. Check Prometheus logs for config reload errors (`/-/reload` returns 200 but bad YAML is rejected).
