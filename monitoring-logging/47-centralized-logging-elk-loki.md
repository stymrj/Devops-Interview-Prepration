# Centralized Logging (ELK / Loki + Promtail) Interview Preparation Guide

*How to Answer Centralized Logging Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why Centralize Logs](#why-centralize-logs)
2. [The ELK / EFK Stack](#the-elk--efk-stack)
3. [Loki and Promtail](#loki-and-promtail)
4. [Logging Strategy and Interview Scenarios](#logging-strategy-and-interview-scenarios)

---

## Why Centralize Logs

### Q1: Why can't we just SSH into servers and tail logs anymore?

**How to Answer:**

"That works when you have five servers — it falls apart completely at fifty, or when everything runs in containers that get rescheduled constantly. A pod can die and take its logs with it before you even SSH anywhere. Centralized logging means every service ships its logs to one searchable place the moment they're written. So when production breaks at 2 AM, I'm searching one query across the whole fleet instead of playing SSH whack-a-mole."

**Key Point:** "Containers are ephemeral — ship logs to one searchable place or lose them when the container dies."

---

### Q2: Walk me through a real log pipeline — what are the moving parts?

**How to Answer:**

"There's always a collector running on each node — Filebeat, Fluent Bit, or Promtail — tailing log files and shipping them off. Those feed into a buffer or processor like Kafka or Logstash that parses and enriches the data. Then everything lands in a store built for search — Elasticsearch or Loki — and you query it through Kibana or Grafana. Four layers: collect, process, store, visualize. When something's slow or dropping logs, I know exactly which layer to check."

**Key Point:** "Collect on the node, process in the middle, store in a search engine, visualize on top — four layers."

---

### Q3: What's the difference between structured and unstructured logging, and which do you enforce?

**How to Answer:**

"I push hard for structured JSON logs in every service I touch. A line like `ERROR payment failed for order 123` is fine for a human but useless for a query — I can't aggregate on it or alert on it reliably. JSON with fields like `level`, `service`, `request_id`, and `latency_ms` means I can filter, count, and graph without regex gymnastics. Unstructured logs still exist in the real world, so I parse them at the collector with patterns — but the goal is always structured at the source."

**Key Point:** "Structured JSON logs with consistent fields beat pretty text lines — queries and alerts need fields, not prose."

---

## The ELK / EFK Stack

### Q4: Why does the ELK stack use Elasticsearch for logs instead of a regular database?

**How to Answer:**

"Because logs are a search problem, not a transaction problem — you need full-text search across billions of lines, not ACID guarantees. Elasticsearch indexes every term in an inverted index, so searching for one error string across a month of logs takes seconds instead of a full table scan. It's also schemaless-friendly, which matters because log formats drift constantly. The tradeoff is that it's resource-hungry — you're paying for that index speed in memory and disk."

**Key Point:** "Logs need full-text search at scale — Elasticsearch's inverted index does that, at the cost of heavy resources."

---

### Q5: What does each piece of the ELK stack actually do?

**How to Answer:**

"Elasticsearch is the search engine and store — it indexes and holds the logs. Logstash is the processing pipeline — it takes raw log lines, parses them with grok patterns, enriches them, and pushes them into Elasticsearch. Kibana is the UI — dashboards, saved searches, and visualizations on top of Elasticsearch. And Beats are the lightweight shippers — Filebeat sits on each node and tails files into Logstash or straight into Elasticsearch. E-L-K plus Beats is the full loop."

**Key Point:** "Beats collect, Logstash processes, Elasticsearch stores and indexes, Kibana visualizes."

---

### Q6: ELK vs EFK — Logstash or Fluentd/Fluent Bit as the processor?

**How to Answer:**

"Logstash is powerful but heavy — it's a JVM process, so it eats memory and you feel it on small nodes. Fluent Bit is the opposite: written in C, tiny footprint, perfect as a DaemonSet on every Kubernetes node. That's why EFK — Elasticsearch, Fluent Bit, Kibana — is the standard Kubernetes logging stack. I'd pick Logstash only if I need its richer filter plugin ecosystem for messy legacy logs. For container logs that are already JSON, Fluent Bit wins on cost and simplicity."

```yaml
# fluent-bit DaemonSet snippet: tail container logs, parse JSON, ship to ES
[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
```

**Key Point:** "Fluent Bit for lightweight Kubernetes shipping, Logstash only when you need its heavier plugin ecosystem."

---

## Loki and Promtail

### Q7: What is Loki and how is it different from Elasticsearch?

**How to Answer:**

"Loki is Grafana's log aggregation system, and its big idea is that it only indexes labels — not the full log content. Elasticsearch indexes every word, which is fast for search but insanely expensive at scale. Loki just stores compressed chunks of logs and indexes metadata like `app=payments, level=error`, so it costs a fraction to run. The catch: searching inside the log text means scanning chunks, so full-text queries are slower. I reach for Loki when cost matters and most queries start from labels."

**Key Point:** "Loki indexes only labels, not full text — far cheaper than Elasticsearch, slightly slower for free-text search."

---

### Q8: How does Promtail actually work on a node?

**How to Answer:**

"Promtail runs as a DaemonSet on every Kubernetes node and tails the container log files the kubelet writes under `/var/log/pods`. For each log line it attaches labels — namespace, pod name, container name — from the file path or Kubernetes metadata. Then it batches and pushes those labeled streams to Loki over HTTP. It's deliberately dumb: no parsing, no indexing, just attach labels and ship. That simplicity is why it's so cheap to run at scale."

```yaml
# promtail config snippet: scrape pod logs, add k8s labels
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

**Key Point:** "Promtail tails pod log files, attaches Kubernetes labels, and pushes streams to Loki — collect and label, nothing more."

---

### Q9: How do you query logs in Loki — what's LogQL?

**How to Answer:**

"LogQL looks a lot like PromQL, which makes it easy if you already know Prometheus. You start with a label selector like `{app='payments', level='error'}` — that's the fast part because labels are indexed. Then you pipe filters: `|= 'timeout'` keeps lines containing timeout, `!= 'healthcheck'` drops noise, and `| json` parses JSON fields so you can filter on them. For counts over time you wrap it: `count_over_time({app='payments'} |= 'error' [5m])`. The rule I follow: filter by labels first, text second — text-only searches scan everything and crawl."

**Key Point:** "Labels first for speed, pipe filters for text — `{app='x'} |= 'error'` is the LogQL pattern to know."

---

### Q10: When would you pick ELK over Loki, and vice versa?

**How to Answer:**

"I'd pick ELK when the team needs heavy full-text search — compliance teams digging through arbitrary log text, or legacy apps with messy unstructured logs. Elasticsearch's full indexing pays off there. I'd pick Loki when the stack is already Prometheus and Grafana, logs are structured, and cost is a real constraint — which honestly describes most startups I've seen. The operational difference is real too: running Elasticsearch well takes a dedicated person, Loki mostly just runs. Interviewers want to hear that it's a cost-vs-search-power tradeoff, not a religious choice."

**Key Point:** "ELK for heavy full-text search and messy logs, Loki for cost-effective structured logging in Prometheus/Grafana shops."

---

## Logging Strategy and Interview Scenarios

### Q11: What's the high-cardinality label trap in Loki, and how do you avoid it?

**How to Answer:**

"This is the classic Loki interview trap. Every unique label combination creates a separate stream, and Loki holds a stream open in memory — so if you label by something like `request_id` or `user_id`, you get millions of streams and Loki falls over. I keep labels to a small bounded set: app, namespace, environment, level, maybe pod name. Everything else — request IDs, trace IDs, user IDs — goes inside the log line as parsed fields, which LogQL can still filter on. Rule of thumb: if a label value is unbounded, it doesn't belong as a label."

**Key Point:** "Unbounded label values like request IDs explode Loki's stream count — keep labels small and bounded, put the rest in the log line."

---

### Q12: How do you keep Elasticsearch from melting down at scale?

**How to Answer:**

"The two things that kill Elasticsearch clusters are unbounded index growth and bad shard counts. I use Index Lifecycle Management: hot indices for the last few days on fast disks, then roll to warm, then delete after the retention window — 30 days is typical. One index per day per log type keeps shard counts predictable, and I size shards around 20-50 GB each. I also cap the fields — dynamic mapping on messy JSON can explode the mapping and take the cluster down. Basically: lifecycle policies, sane shard sizes, and strict mappings."

**Key Point:** "ILM policies for retention, one index per day, shards at 20-50 GB, and locked-down mappings — that's ES survival."

---

### Q13: Scenario — "Searches are suddenly slow and the cluster is yellow." How do you debug?

**How to Answer:**

"Yellow means unassigned replica shards, so first I check if nodes dropped out or disk watermarks tripped — Elasticsearch stops allocating when disks pass 85 percent. I'd run `GET _cluster/allocation/explain` to see exactly why a shard can't assign. If disks are the issue, I check whether ILM actually deleted old indices or if retention silently broke. Slow searches on top of that usually mean too many shards per node or a query hitting cold indices. My fix order: free disk, let shards assign, then look at shard counts and query patterns."

```bash
# first two commands in any ES incident
curl -s 'localhost:9200/_cat/health?v'
curl -s 'localhost:9200/_cluster/allocation/explain?pretty'
```

**Key Point:** "Yellow cluster = unassigned shards — check disk watermarks and allocation explain first, then shard counts."

---

*That's the full guide — 13 questions. Practice the answers out loud, then do the [hands-on lab](labs/47-centralized-logging-elk-loki-lab.md) to make it stick.*
