# Centralized Logging Hands-On Lab

*Spin up a real Loki + Promtail + Grafana stack, ship logs, and query them — 60-90 minutes.*

**Prereqs:** Docker and Docker Compose. Everything runs locally; no cloud account needed.

---

## Exercise 1: Stand up the Loki stack

**Goal:** Get Loki, Promtail, and Grafana running with one command.

**Commands:**
```bash
mkdir loki-lab && cd loki-lab
cat > docker-compose.yml <<'EOF'
services:
  loki:
    image: grafana/loki:3.0.0
    ports: ["3100:3100"]
    command: -config.file=/etc/loki/local-config.yaml
  promtail:
    image: grafana/promtail:3.0.0
    volumes:
      - ./promtail-config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml
  grafana:
    image: grafana/grafana:11.0.0
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
EOF
docker compose up -d
docker compose ps
```

**Expected output:** All three containers show `Up` status.

**Why it matters:** This is the smallest real centralized-logging stack. Interviews love "walk me through setting up Loki" — now you actually have.

---

## Exercise 2: Point Promtail at local logs

**Goal:** Configure Promtail to tail a log file and push to Loki.

**Commands:**
```bash
cat > promtail-config.yml <<'EOF'
server:
  http_listen_port: 9080
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: local
    static_configs:
      - targets: [localhost]
        labels:
          job: applogs
          __path__: /var/log/app/*.log
EOF
mkdir -p /tmp/applog
echo '{"level":"info","service":"checkout","msg":"order placed","order_id":101}' >> /tmp/applog/app.log
# (map /tmp/applog into the promtail container as /var/log/app and restart)
docker compose restart promtail
```

**Expected output:** Promtail restarts cleanly; `docker logs <promtail>` shows no config errors.

**Why it matters:** This is exactly what Promtail does on every Kubernetes node — tail files, attach labels, push streams. The `__path__` + label mechanics are interview gold.

---

## Exercise 3: Add Loki as a Grafana datasource

**Goal:** Wire Grafana to Loki so you can query logs.

**Commands:** Open `http://localhost:3000` → Connections → Add data source → Loki → URL `http://loki:3100` → Save & Test.

**Expected output:** "Data source connected and labels found."

**Why it matters:** The Grafana→Loki link is the "visualize" layer of the pipeline. Every monitoring interview assumes you can do this blindfolded.

---

## Exercise 4: Write your first LogQL queries

**Goal:** Find your log lines using label selectors and pipe filters.

**Commands (in Grafana → Explore → Loki):**
```
{job="applogs"}
{job="applogs"} |= "order"
{job="applogs"} | json | level="info"
count_over_time({job="applogs"} [5m])
```

**Expected output:** First query returns your JSON log lines; the `count_over_time` query draws a small graph.

**Why it matters:** `{labels} |= text | json` is the LogQL pattern behind 80% of real debugging. Muscle memory here pays off in interviews and incidents alike.

---

## Exercise 5: Simulate an error spike and alert on it

**Goal:** Generate error logs and build a Grafana alert on the error rate.

**Commands:**
```bash
for i in $(seq 1 30); do
  echo "{\"level\":\"error\",\"service\":\"checkout\",\"msg\":\"payment timeout\",\"order_id\":$i}" >> /tmp/applog/app.log
  sleep 2
done
```
Then in Grafana → Alerting → New alert rule, Loki datasource, query:
```
sum by (service) (count_over_time({job="applogs"} | json | level="error" [1m]))
```
Set condition: IS ABOVE 5. Evaluate every 1m.

**Expected output:** The alert fires (or goes Pending) within a couple of minutes.

**Why it matters:** Log-based alerting is how teams catch issues metrics miss. This exact pattern — error-rate threshold on parsed JSON logs — comes up constantly in SRE interviews.

---

## Exercise 6: Add Kubernetes-style labels with relabeling

**Goal:** See how Promtail turns file paths into useful labels.

**Commands:** Edit `promtail-config.yml`, add a pipeline stage:
```yaml
pipeline_stages:
  - json:
      expressions:
        level: level
        service: service
  - labels:
      level:
      service:
```
Restart Promtail, write a few new log lines, then query `{service="checkout"}` in Grafana.

**Expected output:** `{service="checkout"}` returns only checkout logs — the JSON field became a real indexed label.

**Why it matters:** This is the labels-vs-fields decision from the guide made concrete. You just promoted fields to labels — and you can feel why you'd only do it for bounded values.

---

## Exercise 7: Break it — high-cardinality labels

**Goal:** Experience the cardinality trap firsthand (safely).

**Commands:**
```bash
# DON'T do this in production — watch what happens locally instead
# Add order_id as a label in pipeline_stages, restart, then:
for i in $(seq 1 200); do
  echo "{\"level\":\"info\",\"service\":\"checkout\",\"msg\":\"ok\",\"order_id\":$i}" >> /tmp/applog/app.log
done
```
Check Loki's memory: `docker stats loki-lab-loki-1` and query `count by (__stream_shard__) ({job="applogs"})` — or just note the stream count exploding in Loki's `/loki/api/v1/label/__name__/values` slowness.

**Expected output:** Noticeably higher memory/stream churn for zero querying benefit — then revert the config.

**Why it matters:** Interviewers ask about the cardinality trap because it's the #1 way people break Loki. Now you've seen the failure mode instead of just memorizing the warning.

---

## Exercise 8: Ship the same logs to Elasticsearch (EFK taste)

**Goal:** Compare the Elasticsearch experience — full-text indexing, heavier footprint.

**Commands:**
```bash
docker run -d --name es -p 9200:9200 -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" docker.elastic.co/elasticsearch/elasticsearch:8.14.0
sleep 30
curl -s localhost:9200/_cat/health
curl -s -X POST localhost:9200/logs/_doc -H 'Content-Type: application/json' \
  -d '{"level":"error","service":"checkout","msg":"payment timeout","order_id":42}'
curl -s -X POST localhost:9200/logs/_search -H 'Content-Type: application/json' \
  -d '{"query":{"match":{"msg":"timeout"}}}' | head -c 400
```

**Expected output:** Health `green` (single node), and the search returns your document — full-text `match` on `msg` with no label setup.

**Why it matters:** Feel the difference: ES indexed and searched the raw text with zero label planning, but check `docker stats es` — compare its memory to Loki's. That's the cost-vs-search tradeoff in one screen.

---

## Exercise 9: Retention — set a short TTL and watch ILM thinking

**Goal:** Practice the retention mindset from the guide.

**Commands:**
```bash
# Elasticsearch: delete-by-age thinking (manual version of ILM)
curl -s -X PUT localhost:9200/logs/_settings -H 'Content-Type: application/json' \
  -d '{"index.lifecycle.name": "30day-policy"}' | head -c 200
# Loki: check the retention config knob
docker exec loki-lab-loki-1 cat /etc/loki/local-config.yaml | grep -A3 retention
```

**Expected output:** ES accepts the lifecycle setting; Loki shows its default `retention_period` (often 744h).

**Why it matters:** Retention is where logging budgets live or die. Knowing the knobs — ILM policies vs Loki's `retention_period` — is a standard senior-level interview question.

---

## Exercise 10: Tear down and write your runbook

**Goal:** Turn today's work into interview ammunition.

**Commands:**
```bash
docker compose down -v
docker rm -f es
```

Then write 5 bullet points in your own words: (1) the four pipeline layers, (2) when you'd pick Loki vs ES, (3) the cardinality rule, (4) your LogQL go-to query, (5) the first two commands in an ES-yellow incident.

**Expected output:** A half-page runbook you can recite.

**Why it matters:** The lab only counts if you can talk about it. These five bullets cover the questions interviewers actually ask from this topic.
