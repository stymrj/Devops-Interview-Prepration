# Hands-On Lab: Grafana Dashboards & Alerting

9 exercises. Goal: run Grafana locally, connect it to Prometheus, build a real dashboard with variables, and wire an end-to-end alert.

**Prerequisites:** Docker installed. Exercises 1–8 run on your laptop; no Kubernetes needed.

---

## Exercise 1: Run Grafana with Docker

**Goal:** Start Grafana and log in.

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana-oss
```

**Expected output:** http://localhost:3000 shows the login page; `admin` / `admin` gets you in (it forces a password change).
**Why it matters:** A local Grafana is the fastest sandbox for dashboard and alert experiments.

---

## Exercise 2: Add Prometheus as a datasource

**Goal:** Connect Grafana to a Prometheus backend.

```bash
docker run -d --name prometheus -p 9090:9090 prom/prometheus
```

Then in Grafana: Configuration → Data sources → Add → Prometheus → URL `http://host.docker.internal:9090` (or `http://prometheus:9090` if on a shared Docker network) → Save & Test.

**Expected output:** "Data source is working" green banner.
**Why it matters:** Every panel queries through a datasource — a broken datasource is the #1 cause of "dashboard shows no data."

---

## Exercise 3: Build your first panel

**Goal:** Graph Prometheus's own scrape health.

New dashboard → Add panel → pick the Prometheus datasource, and enter:

```promql
up
```

Set the visualization to Time series, give the panel the title "Target health".

**Expected output:** A graph showing `up == 1` lines for each scrape target.
**Why it matters:** This is the smallest possible proof that query → panel → dashboard works end to end.

---

## Exercise 4: Add a template variable

**Goal:** Make the dashboard reusable with a `$job` dropdown.

Dashboard settings → Variables → Add variable: name `job`, type Query, datasource Prometheus, query `label_values(up, job)`. Then change your panel query to:

```promql
up{job="$job"}
```

**Expected output:** A dropdown appears at the top of the dashboard; picking a job filters the panel.
**Why it matters:** Variables are how one dashboard serves every service instead of cloning dashboards per team.

---

## Exercise 5: Build a RED row

**Goal:** Add rate / errors / duration panels for a service.

```promql
# Request rate
sum(rate(prometheus_http_requests_total[5m]))
# Error rate
sum(rate(prometheus_http_requests_total{code=~"5.."}[5m]))
```

**Expected output:** Three panels showing traffic, errors, and request volume for Prometheus itself.
**Why it matters:** RED (rate, errors, duration) is the standard first row of any service dashboard — interviewers expect you to know it.

---

## Exercise 6: Create a Grafana alert rule

**Goal:** Fire an alert when a target goes down.

Alerting → Alert rules → New: query `up == 0`, condition "last value below 1", evaluate every `1m`, for `2m`. Set a label `severity=critical`.

**Expected output:** The rule shows as Normal; stop the Prometheus container and watch it go Pending → Firing within ~3 minutes.
**Why it matters:** This is the exact evaluate → pending → firing pipeline interviewers ask you to walk through.

---

## Exercise 7: Route the alert to a contact point

**Goal:** Send the firing alert somewhere visible.

Alerting → Contact points → add a Slack or webhook contact point (use a webhook.site URL for testing). Then Alerting → Notification policies → add a nested policy matching `severity=critical` → route to your contact point.

**Expected output:** Stopping Prometheus again delivers a JSON payload to your webhook URL.
**Why it matters:** Alerts that fire but route nowhere are the classic production gap — routing by labels is the fix.

---

## Exercise 8: Add a mute timing for maintenance

**Goal:** Silence alerts during a known maintenance window.

Alerting → Notification policies → Mute timings → create `db-maintenance` for Saturday 02:00–04:00, then attach it to the notification policy.

**Expected output:** Alerts matching the policy during that window are suppressed but still visible as muted in the alert list.
**Why it matters:** Mute timings stop the "we knew about the deploy, why did it page" problem.

---

## Exercise 9: Export the dashboard as JSON

**Goal:** Save the dashboard to git.

Dashboard settings → JSON Model → copy, and save to `dashboards/service-health.json` in a git repo. Bonus: mount it back via provisioning:

```yaml
# provisioning/dashboards/default.yaml
providers:
  - name: default
    options:
      path: /etc/grafana/provisioning/dashboards
```

**Expected output:** The dashboard JSON is committed; restarting Grafana with the provisioned path reloads it automatically.
**Why it matters:** Dashboards as code — reviewed in PRs, reproducible across environments — is the professional standard.

---

*Cleanup: `docker rm -f grafana prometheus` when done.*
