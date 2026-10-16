# Grafana Dashboards & Alerting Interview Preparation Guide

*How to Answer Grafana Dashboards & Alerting Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Grafana Core Concepts](#grafana-core-concepts)
2. [Building Dashboards That Actually Help](#building-dashboards-that-actually-help)
3. [Alerting Done Right](#alerting-done-right)

---

## Grafana Core Concepts

### Q1: What is Grafana and where does it fit in the monitoring stack?

**How to Answer:**

"I think of Grafana as the visualization and alerting layer — it doesn't collect metrics itself, it queries whatever backend holds them. Prometheus, Loki, CloudWatch, Elasticsearch — Grafana sits on top and turns their data into dashboards and alerts. That separation is the whole point: you can swap the backend without rebuilding every dashboard. In most Kubernetes setups I've worked with, it's Prometheus doing the scraping and Grafana doing the showing."

**Key Point:** "Grafana visualizes and alerts on data it queries from backends like Prometheus — it stores no metrics itself."

---

### Q2: How do datasources, panels, and dashboards relate?

**How to Answer:**

"A datasource is just a configured connection to a backend — Prometheus, Loki, whatever — with its auth and query language. A panel is one visualization on that data: a graph, a stat, a table, each running its own query against a datasource. A dashboard is a collection of panels with layout, time range, and shared variables. So when someone says 'the dashboard is broken,' I start by asking which panel and which datasource — nine times out of ten it's the query or the datasource connection, not Grafana itself."

**Key Point:** "Datasource = backend connection, panel = one query visualized, dashboard = panels plus layout and shared time range."

---

### Q3: How do you manage dashboards as code in a real team?

**How to Answer:**

"I never let dashboards live only in the Grafana UI — they get exported as JSON and committed to git. The standard path is provisioning: you drop dashboard JSON into a directory Grafana watches, and it loads them on startup, no clicking around. For Kubernetes I use the Grafana Operator or the kube-prometheus-stack Helm chart, where dashboards ride along as ConfigMaps. The payoff is obvious — dashboard changes get reviewed in PRs, they survive Grafana reinstalls, and every environment shows the same dashboards."

```yaml
# grafana-values.yaml (kube-prometheus-stack)
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      providers:
        - name: default
          options:
            path: /var/lib/grafana/dashboards/default
```

**Key Point:** "Export dashboards as JSON, commit to git, load via provisioning — dashboards become reviewed, reproducible artifacts."

---

### Q4: How does Grafana handle auth, orgs, and team access in production?

**How to Answer:**

"For auth I wire Grafana to the company's SSO — usually OIDC or SAML — so there's no separate Grafana password to manage, and role mapping comes from IdP groups. Orgs are Grafana's hard isolation boundary: separate datasources, dashboards, and users per org, which I use when teams truly shouldn't see each other's stuff. Inside one org, folders plus RBAC roles — Viewer, Editor, Admin — control who can touch what. The trap is giving everyone Editor: then someone edits a shared dashboard at 2am and nobody knows what changed."

**Key Point:** "SSO for auth, orgs for hard isolation, folders and RBAC roles for day-to-day access — don't hand out Editor to everyone."

---

## Building Dashboards That Actually Help

### Q5: How do you design a dashboard that's actually useful during an incident?

**How to Answer:**

"I build incident dashboards top-down: the first row answers 'is the service healthy' — error rate, latency p95, traffic, saturation. Below that, the drill-down rows: per-endpoint, per-pod, per-dependency, so I can go from 'checkout is erroring' to 'it's the payments pod on node 3' in two clicks. I use the RED method for services — rate, errors, duration — and USE for resources — utilization, saturation, errors. And I keep the panel count low: a 40-panel dashboard is a wall of noise nobody reads at 3am."

**Key Point:** "Top row = service health, below = drill-downs; RED for services, USE for resources, and keep it readable at 3am."

---

### Q6: What are template variables and why do interviewers keep asking about them?

**How to Answer:**

"Template variables turn one dashboard into many — instead of a dashboard per service, I have a `$service` dropdown and the same panels re-query for whichever service is picked. They're just variables interpolated into queries, usually populated from a label-values query against Prometheus. The classic demo: a `$namespace` variable that drives every panel, so the SRE onboards a new namespace by selecting it, not by cloning dashboards. Interviewers ask because it separates people who click around from people who build reusable observability."

```promql
# inside a panel, $service comes from the dashboard variable
sum(rate(http_requests_total{service="$service", status=~"5.."}[5m]))
```

**Key Point:** "Variables parameterize dashboards — one dashboard serves every service or namespace instead of cloning per team."

---

### Q7: How do you keep dashboards fast and stop dashboard sprawl?

**How to Answer:**

"Slow dashboards are almost always slow queries — I check the query inspector for the heaviest panel, then fix it with recording rules or narrower time ranges instead of letting every panel scan raw series. I also cap the default time range and refresh interval; a 30-day range with 5-second refresh on a busy cluster will melt the browser and the backend. For sprawl I set a rule: dashboards live in team folders, get reviewed, and unused ones get deleted — a dashboard nobody opened in 90 days is clutter, not observability."

**Key Point:** "Fix slow dashboards at the query level with recording rules and sane defaults; delete dashboards nobody opens."

---

## Alerting Done Right

### Q8: How does Grafana alerting actually evaluate and fire?

**How to Answer:**

"Grafana alerting runs your query on an interval — say every minute — and checks the result against a threshold condition, like error rate above 5%. An alert isn't instant: it moves from Normal to Pending while the condition holds, and only fires after the 'for' duration passes, which filters out one-minute blips. Once firing, it goes to a notification policy that decides who gets paged based on labels. So the full path is: query evaluated on interval, condition held for the 'for' window, then routed by labels to a contact point."

**Key Point:** "Evaluate on an interval, require the condition to hold through the 'for' window, then route by labels — that's the whole pipeline."

---

### Q9: Grafana alerting vs Alertmanager — when do you use which?

**How to Answer:**

"They solve the same problem at different layers. Prometheus Alertmanager handles alerts defined in Prometheus recording and alerting rules — it's the classic path if Prometheus is your only datasource. Grafana alerting is datasource-agnostic: one alerting engine across Prometheus, Loki, CloudWatch, even SQL databases, with a UI for building rules instead of YAML. I reach for Alertmanager when the stack is pure Prometheus and the team already lives in YAML. I reach for Grafana alerting when alerts need to span multiple backends — like paging on a Loki log pattern and a Prometheus metric from one rule set."

**Key Point:** "Alertmanager for pure-Prometheus YAML workflows; Grafana alerting when rules must span multiple datasources."

---

### Q10: What makes an alert good versus a 3am pager that everyone ignores?

**How to Answer:**

"A good alert is actionable, urgent, and tied to user impact — 'checkout error rate above 2% for 10 minutes, runbook linked' tells me what's broken and what to do. A bad alert fires on a symptom nobody can act on, like 'CPU at 85%,' with no runbook and no customer impact. I alert on symptoms from the user's perspective — error budget burn, latency, availability — not on internal causes. And every alert needs an owner and a runbook link, otherwise it's just noise that trains the team to ignore pages."

**Key Point:** "Alert on user-facing symptoms with an owner and runbook — 'CPU at 85%' with no action is noise, not an alert."

---

### Q11: How do you route alerts to the right person or team?

**How to Answer:**

"Routing is all labels. Every alert carries labels like team, service, and severity, and notification policies match on those labels to pick the contact point — PagerDuty for the payments team, Slack for the platform team's warnings. I set up a default policy as a catch-all so nothing silently drops, then nested policies for team-specific routing. Mute timings handle the known stuff — maintenance windows, deploys — so alerts don't fire during a planned database migration. The interview trap is routing by alert name instead of labels; names change, label conventions don't."

```yaml
# notification policy sketch: route by labels, not alert names
routes:
  - matchers: [team = payments, severity = critical]
    receiver: pagerduty-payments
  - matchers: [severity = warning]
    receiver: slack-platform-warnings
```

**Key Point:** "Route on labels — team, service, severity — with a catch-all default and mute timings for planned work."

---

### Q12: How do you fight alert fatigue in practice?

**How to Answer:**

"I treat every page as a bug report against the alerting config. If an alert fires and nobody does anything, I either fix the threshold, add a 'for' window, or delete the alert — alerting on it again unchanged is just choosing to be woken up. I deduplicate aggressively: one alert per incident, grouped by service, not ten panels screaming about the same outage. And I track alert volume per team per week — if a team gets more than a couple of pages a week, their thresholds are wrong, not their luck. The goal is that when the pager goes off, everyone believes it's real."

**Key Point:** "Unactionable alerts get fixed or deleted, incidents get one grouped page, and page volume per team is a metric you watch."

---

*Built for interview prep, one deep-dive at a time.*
