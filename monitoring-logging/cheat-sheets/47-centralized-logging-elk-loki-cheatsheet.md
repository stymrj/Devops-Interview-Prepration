# Centralized Logging Cheat Sheet

*One-page dense reference — LogQL, Promtail, Elasticsearch, Fluent Bit.*

## LogQL (Loki)

```
{app="payments", env="prod"}            # label selector (indexed, fast)
{app="payments"} |= "timeout"          # line contains
{app="payments"} != "healthcheck"      # line does NOT contain
{app="payments"} |~ "5[0-9]{2}"        # regex match
{app="payments"} | json                # parse JSON fields
{app="payments"} | json | level="error"  # filter on parsed field
{app="payments"} | logfmt               # parse logfmt
count_over_time({app="x"} |= "error" [5m])       # error count trend
sum by (app) (rate({env="prod"} [5m]))            # log rate per app
topk(5, sum by (pod) (count_over_time({ns="prod"}[1h])))  # noisiest pods
```

**Rule:** labels first (fast), `|=` text filters second (scans chunks).

## Promtail config essentials

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs: [{role: pod}]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
pipeline_stages:
  - json: {expressions: {level: level}}   # parse
  - labels: {level:}                      # promote BOUNDED fields only
```

**Never label:** request_id, user_id, trace_id, timestamps — unbounded = stream explosion.

## Elasticsearch ops

```bash
curl localhost:9200/_cat/health?v                      # cluster health
curl localhost:9200/_cat/indices?v                     # index list + sizes
curl localhost:9200/_cat/shards?v | grep UNASSIGNED   # stuck shards
curl 'localhost:9200/_cluster/allocation/explain?pretty'
# yellow cluster first checks: disk watermarks (85%/90%), dead nodes
```

```json
// full-text search
POST logs/_search
{"query": {"match": {"msg": "timeout"}}}
// structured filter + agg
POST logs/_search
{"query": {"bool": {"filter": [
  {"term": {"level": "error"}},
  {"range": {"@timestamp": {"gte": "now-1h"}}}]}},
 "aggs": {"per_svc": {"terms": {"field": "service.keyword"}}}}
```

**ILM survival:** 1 index/day per log type · shards 20–50 GB · hot→warm→delete (30d typical) · lock down dynamic mappings.

## Fluent Bit (EFK on K8s)

```
[INPUT]  Name tail  Path /var/log/containers/*.log  Parser docker
[FILTER] Name kubernetes  Match kube.*      # enrich with k8s metadata
[OUTPUT] Name es  Match *  Host elasticsearch  Port 9200  Index fluent-bit
```

Parsers: `docker` / `cri` for container JSON · multiline filters for stack traces (Java/Python).

## Loki vs Elasticsearch — 10-second pick

| Need | Pick |
|---|---|
| Cheap, structured logs, Grafana shop | Loki |
| Full-text search, messy legacy logs | Elasticsearch |
| Compliance deep-dives on raw text | Elasticsearch |
| 1–3 person ops team | Loki |
| Already on Prometheus | Loki (same query language family) |

## Interview one-liners

- "Loki indexes labels, not text — cheaper, slower free-text search."
- "Promtail: tail files, attach labels, push — deliberately dumb."
- "Yellow ES cluster → `allocation/explain` → check disk watermarks first."
- "Unbounded label values kill Loki. Bounded labels only."
- "Structured JSON at the source beats parsing at the collector."
