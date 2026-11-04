# Observability Cost & Cardinality — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Cardinality | Unique label combinations — it multiplies, never adds |
| Safe label | Bounded values you control: method, status, service, env, AZ |
| Dangerous label | Unbounded or user-controlled: user ID, request ID, IP, full URL with IDs |
| Series explosion | One bad label turning one metric into millions of series |
| Ingestion cost | What you pay to send and store data |
| Retention cost | What you pay to keep it — raw for days, rollups for months |
| Query cost | What heavy dashboards and alert evaluation add on top |
| Exemplar | Trace ID attached to a metric point, linking spike to exact requests |
| Head sampling | Blind keep-one-in-N at the start — drops interesting traces too |
| Tail sampling | Keep errors and slow traces after they finish |
| Recording rule | Precompute a slow, frequent query into a cheap series |
| Downsampling | Keep lower resolution for old data, throw the rest away |
| Relabeling | Drop or rewrite labels at ingestion, before series are created |

## Label safety at a glance

| Label | Safe? | Why |
|---|---|---|
| method, status | Yes | Tiny, bounded sets |
| service, environment | Yes | Controlled by you |
| path template (/users/:id) | Yes | Bounded by your routes |
| path with IDs (/users/123) | No | One series per ID |
| user_id, email, session_id | No | Unbounded, user-controlled |
| request_id, IP address | No | New value per request |
| error message with dynamic text | No | New value per error instance |

## Cost drivers, in check order

1. Ingested volume — how much you send
2. Retention — how long you keep it
3. Cardinality — how many series metrics create
4. Query / alert evaluation load — how often heavy queries run

## Fix order for a cardinality incident

1. Emergency relabeling — drop the bad label, stop new series
2. Source fix — normalize paths to route templates in the app
3. Move detail — per-entity detail into logs/traces, linked by exemplar
4. Guardrail — series-count alert + CI check on new labels

## Retention tiers

| Tier | Keep for | For |
|---|---|---|
| Raw, high-resolution | Days | Live debugging |
| Hourly rollups / downsampled | Months | Trend questions |
| Audit / compliance archive | As required, in object storage | Compliance only |

## Sampling: which, when

| Approach | Keeps | Use when |
|---|---|---|
| Head sampling | Random 1-in-N | Volume reduction matters more than completeness |
| Tail sampling | Errors, slow traces | Errors are rare but expensive to miss |

## Guardrails checklist

- [ ] Alert on series-count growth
- [ ] Dashboard of cost per team / service
- [ ] CI check failing on new high-cardinality labels
- [ ] Monthly review of top 10 metrics by series and cost
- [ ] Recording rules only for slow, frequently used queries
