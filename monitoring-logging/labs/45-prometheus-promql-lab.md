# Hands-On Lab: Prometheus & PromQL

10 exercises. Goal: run Prometheus locally, scrape real targets, write PromQL that actually answers questions, and wire an alert end to end.

**Prerequisites:** Docker installed. No Kubernetes needed for most of this lab — exercises 1–9 run on your laptop.

---

## Exercise 1: Run Prometheus with Docker

**Goal:** Start Prometheus with a custom config.

```bash
mkdir -p prom-lab && cd prom-lab
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF
docker run -d --name prometheus -p 9090:9090 \
  -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml" \
  prom/prometheus
```

**Expected output:** `docker ps` shows prometheus running; http://localhost:9090 shows the Prometheus UI.
**Why it matters:** This is the fastest way to get a real Prometheus to experiment against.

---

## Exercise 2: Scrape a real target (node_exporter)

**Goal:** Add a second target and watch it appear in the Targets page.

```bash
docker run -d --name node-exporter -p 9100:9100 prom/node-exporter
cat >> prometheus.yml <<'EOF'
  - job_name: 'node'
    static_configs:
      - targets: ['host.docker.internal:9100']
EOF
docker restart prometheus
```

Open **Status → Targets** in the UI — both jobs should show UP.

**Expected output:** Two targets, both state UP.
**Why it matters:** The Targets page is your first stop when a metric is missing — DOWN here means check the scrape, not the app.

---

## Exercise 3: Run your first PromQL queries

**Goal:** Query instant and range vectors in the expression browser.

```promql
up
up{job="node"}
node_cpu_seconds_total[5m]
```

**Expected output:** `up` returns one series per target with value 1; the `[5m]` query shows a range vector (graph tab draws it).
**Why it matters:** Instant vs range vectors is the single most-tested PromQL concept in interviews.

---

## Exercise 4: Compute rates with rate()

**Goal:** Get per-second CPU usage from a counter.

```promql
rate(node_cpu_seconds_total{mode="idle"}[5m])
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

**Expected output:** Values between 0 and 1 — the second query is overall CPU utilization.
**Why it matters:** You almost never graph a raw counter; rate() turns it into something meaningful. Try `irate()` on the same query and notice how much jumpier it is.

---

## Exercise 5: Aggregate with by and topk

**Goal:** Find the busiest CPU mode across the machine.

```promql
topk(3, sum by (mode) (rate(node_cpu_seconds_total[5m])))
```

**Expected output:** Three rows — usually `idle`, `iowait`/`system`, `user` with their per-second rates.
**Why it matters:** Aggregation is how you go from 50 series to the one number an alert or dashboard needs.

---

## Exercise 6: Write a recording rule

**Goal:** Precompute CPU utilization into a new series.

```bash
cat > rules.yml <<'EOF'
groups:
  - name: node-rules
    interval: 30s
    rules:
      - record: node:cpu_utilization:rate5m
        expr: 1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))
EOF
```

Mount it into Prometheus (`--web.enable-lifecycle` helps reload):

```bash
docker run -d --name prometheus -p 9090:9090 \
  -v "$PWD/prometheus.yml:/etc/prometheus/prometheus.yml" \
  -v "$PWD/rules.yml:/etc/prometheus/rules.yml" \
  prom/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.enable-lifecycle
# then add to prometheus.yml:
# rule_files: ['/etc/prometheus/rules.yml']
curl -X POST http://localhost:9090/-/reload
```

Query `node:cpu_utilization:rate5m` — it's now a plain series.

**Expected output:** The recorded series appears in the expression browser.
**Why it matters:** Recording rules turn expensive dashboard queries into cheap single-series lookups.

---

## Exercise 7: Fire a real alert

**Goal:** Create an alerting rule and watch it go Pending → Firing.

```bash
cat > alerts.yml <<'EOF'
groups:
  - name: node-alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels: { severity: warning }
        annotations: { summary: "Instance {{ $labels.instance }} down" }
EOF
```

Add `alerts.yml` to `rule_files`, reload, then stop node_exporter (`docker stop node-exporter`) and watch **Alerts** page.

**Expected output:** Alert shows Pending, then Firing after 1 minute.
**Why it matters:** This is the full alerting loop — rule evaluation, `for` duration, state transitions — minus Alertmanager routing.

---

## Exercise 8: Instrument a tiny app

**Goal:** Expose a counter from your own code and scrape it.

```python
# app.py
from prometheus_client import Counter, start_http_server
import time, random

REQUESTS = Counter('demo_requests_total', 'Demo requests', ['status'])

start_http_server(8000)
while True:
    REQUESTS.labels(status=random.choice(['200', '200', '500'])).inc()
    time.sleep(1)
```

```bash
pip install prometheus_client
python3 app.py &
# add to prometheus.yml:
#   - job_name: 'demo'
#     static_configs:
#       - targets: ['host.docker.internal:8000']
curl -X POST http://localhost:9090/-/reload
```

**Expected output:** `demo_requests_total` queryable in Prometheus; `rate(demo_requests_total[5m])` shows ~1/sec.
**Why it matters:** Instrumentation is half the job — knowing how a counter gets born makes PromQL click.

---

## Exercise 9: Compute an error rate like an SRE

**Goal:** Build the classic error-rate query on your demo app.

```promql
sum by (status) (rate(demo_requests_total[5m]))
sum(rate(demo_requests_total{status="500"}[5m]))
  / sum(rate(demo_requests_total[5m]))
```

**Expected output:** Per-status rates, then an error ratio around 0.33.
**Why it matters:** This exact query shape — bad/total over a window — is the backbone of every error-budget alert you'll ever write.

---

## Exercise 10: Break it and debug the scrape

**Goal:** Practice the most common Prometheus troubleshooting loop.

1. Stop the demo app (`kill %1`) and watch the target go DOWN.
2. Check **Status → Targets** — note the error message ("connection refused").
3. Restart the app with a wrong port in `prometheus.yml`, reload, read the new error.
4. Fix the port, reload, confirm UP.

**Expected output:** You can narrate exactly where each failure showed up.
**Why it matters:** In interviews, "how do you debug a missing metric" is a top-5 question — the answer is always: Targets page first, then config, then the app.
