# Cheat Sheet: Grafana Dashboards & Alerting

## Core concepts

- Grafana = visualization + alerting layer; stores no metrics, queries backends (datasources)
- Datasource → backend connection (Prometheus, Loki, CloudWatch…) with its own query language
- Panel → one query, one visualization (time series, stat, table, gauge…)
- Dashboard → panels + layout + shared time range + template variables
- Org → hard isolation boundary (separate datasources, dashboards, users)
- Folder + RBAC (Viewer / Editor / Admin) → day-to-day access control inside an org

## Dashboard design

- First row: service health — error rate, latency p95, traffic, saturation
- Below: drill-down rows (per endpoint, per pod, per dependency)
- RED for services: Rate, Errors, Duration — USE for resources: Utilization, Saturation, Errors
- Template variable: `$name` interpolated into queries, populated via `label_values(metric, label)`
- Keep panels few and fast; a 40-panel dashboard is noise at 3am

## Useful queries (Prometheus datasource)

```promql
# Error rate (5xx) for a service
sum(rate(http_requests_total{service="$service", status=~"5.."}[5m]))
# p95 latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service="$service"}[5m])) by (le))
# Populate a $job variable
label_values(up, job)
# Target health
up == 0
```

## Alerting pipeline

1. Query evaluated on an interval (e.g. every 1m)
2. Condition checked against threshold (e.g. error rate > 5%)
3. State: Normal → Pending (holds through `for` duration) → Firing
4. Notification policy routes by labels (team, service, severity) → contact point
5. `for` window filters one-minute blips; alert on symptoms, not causes

## Alerting vs Alertmanager

- Alertmanager: Prometheus-native, YAML rules, pure-Prometheus stacks
- Grafana alerting: datasource-agnostic (Prometheus + Loki + CloudWatch…), UI-built rules
- Route on labels, never on alert names; always keep a catch-all default policy
- Mute timings: suppress during maintenance windows / deploys (alerts still visible as muted)

## Good alert checklist

- [ ] Tied to user impact (error budget burn, latency, availability)
- [ ] Actionable — the receiver knows what to do
- [ ] Has an owner (team label) and a runbook link
- [ ] `for` window set — no single-spike pages
- [ ] If it fired and nobody acted → fix the threshold or delete the alert

## Dashboards as code

- Export: dashboard settings → JSON Model → commit to git
- Provisioning: drop JSON in a watched dir, Grafana loads on startup
- Kubernetes: Grafana Operator or kube-prometheus-stack Helm chart (dashboards as ConfigMaps)
- Never let dashboards live only in the UI

## Auth quick reference

- Production auth: OIDC / SAML SSO with IdP group → Grafana role mapping
- Don't hand Editor to everyone — shared dashboards get edited at 2am with no audit trail
- Service accounts + tokens for automation, not personal API keys

## Interview one-liners

- "Grafana queries backends; it stores nothing itself."
- "Variables turn one dashboard into many."
- "Alert on user-facing symptoms with an owner and a runbook."
- "Route on labels, keep a catch-all, mute during planned work."
- "An alert nobody acts on is a bug in the alerting config."
