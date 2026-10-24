# Progressive Delivery — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## Core picture

| Piece | Job |
|-------|-----|
| Canary | Small slice of real traffic on the new version, stepped up on evidence |
| Blue-green | Two full environments, traffic flips in one move — instant rollback |
| Rolling | Pods swap gradually — simple, but no side-by-side comparison |
| Analysis | Metric checks that decide promote vs abort, run at each step |
| Stable service | Old version kept alive until the canary fully wins |

## Strategy picker

- Canary: risky, user-facing releases — you want real traffic evidence
- Blue-green: you need instant rollback and can pay double capacity
- Rolling: simple internal services where comparison matters less

## Argo Rollouts (workload-side)

- **Rollout** resource replaces the Deployment: canary service + stable service + steps
- Steps: `setWeight` -> `pause` -> `analysis` -> next weight, up to 100
- **AnalysisTemplate** = pass/fail bar in Git (e.g. error rate < 1% from Prometheus)
- **AnalysisRun** = the live measurement; a failed run aborts the rollout
- Abort/rollback = traffic back to stable, canary scales down — no redeploy
- `kubectl argo rollouts get/promote/abort rollout <name>`

## Flagger (mesh-side)

- **Canary** resource points at an existing Deployment — Deployment stays untouched
- Provider: Istio, Linkerd, NGINX, App Mesh, Traefik — traffic shifts at that layer
- Analysis: `interval`, `threshold`, `maxWeight`, `stepWeight`, metric thresholds
- Webhooks: load test before promoting, Slack alert on failure
- Failed checks past threshold -> traffic back, canary scaled to zero

## Metrics that decide

- Canary-only (never blended): error rate, p99 latency, request success rate
- Business signals too: checkout success, not just HTTP 200s
- Thresholds set before the rollout, never invented after it looks bad

## Debug checklist

```bash
kubectl argo rollouts get rollout <name> -n <ns>  # step, weight, analysis state
kubectl get endpoints <canary-svc>                 # canary actually receiving traffic?
kubectl get analysisrun -n <ns>                    # what did the analysis measure?
kubectl argo rollouts abort <name> -n <ns>         # safe manual abort path
```

Classic bug: canary gets zero traffic, analysis passes with zero samples — green dashboard, broken release.

## Lines that land

- "CI/CD ships the version. Progressive delivery controls exposure."
- "Rollback is traffic, not a redeploy — the old version stays alive until the canary wins."
- "Deployment moves containers, the rollout controller moves risk."
- "A green dashboard with a failing canary usually means a wrong query, blended metrics, or a canary with no traffic."

## Red flags to avoid

- Promoting on blended metrics instead of canary-only numbers
- Retrying a failed canary by hand instead of letting it abort
- Setting thresholds after the rollout already looks bad
- Deleting the stable version before the canary reaches 100%
- Calling a slow rolling deploy a canary — a canary is a measured experiment
