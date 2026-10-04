# Deployments & Rollout Strategies — Cheat Sheet

Topic 22 of 58. Dense reference for interviews and on-call.

## Create and manage

```bash
kubectl create deploy web --image=nginx:1.25 --replicas=3
kubectl scale deploy web --replicas=5
kubectl autoscale deploy web --min=3 --max=10 --cpu-percent=70
kubectl delete deploy web
```

## Inspect

```bash
kubectl get deploy                        # all Deployments
kubectl describe deploy web               # Conditions + Events
kubectl get deploy web -o yaml            # full spec
kubectl get rs                            # ReplicaSets it owns
```

## Updates

```bash
kubectl set image deploy/web nginx=nginx:1.26      # rolling update
kubectl edit deploy web                            # edit template in place
kubectl apply -f deploy.yaml                       # declarative update
kubectl rollout status deploy/web                  # watch progress
```

## Rollout lifecycle

```bash
kubectl rollout history deploy/web                 # list revisions
kubectl rollout history deploy/web --revision=3    # inspect one revision
kubectl rollout undo deploy/web                    # rollback one step
kubectl rollout undo deploy/web --to-revision=2    # rollback to a revision
kubectl rollout pause deploy/web                   # freeze mid-update
kubectl rollout resume deploy/web                  # continue
kubectl rollout restart deploy/web                 # restart all Pods (new RS)
```

## Strategy spec

```yaml
strategy:
  type: RollingUpdate              # or Recreate
  rollingUpdate:
    maxUnavailable: 0              # pods allowed down (number or %)
    maxSurge: 1                    # extra pods allowed above replicas
```

- Defaults: `maxUnavailable: 25%`, `maxSurge: 25%`
- Zero downtime: `maxUnavailable: 0`
- Tight on resources: `maxSurge: 0` (kill-before-create)

## Revision control

```yaml
spec:
  revisionHistoryLimit: 10   # old ReplicaSets kept for rollback (default 10)
```

- Only Pod template changes create revisions (image, env, probes)
- `kubectl.kubernetes.io/change-cause` annotation documents why (set with `kubectl annotate`)

## Readiness probe (rollout safety)

```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

- New Pods get traffic only after probe passes
- Bad probe → rollout stalls, old Pods stay up (fails safe)

## Stuck rollout triage

```bash
kubectl rollout status deploy/web            # what it's waiting on
kubectl get pods -l app=web                   # CrashLoopBackOff? Pending?
kubectl describe pod <pod>                    # Events: image pull, probes
kubectl logs <pod>                            # app errors
kubectl get events --sort-by=.lastTimestamp   # cluster-wide view
kubectl rollout undo deploy/web               # recover first, debug after
```

## Blue-green (manual, via Service selector)

```bash
# green Deploy already running, verified:
kubectl patch svc web-svc -p '{"spec":{"selector":{"app":"web","version":"green"}}}'
# rollback: switch selector back to "blue"
```

## Canary (no mesh)

- Two Deployments, one Service selecting both labels
- Replica ratio = traffic ratio (e.g. 9 stable : 1 canary ≈ 10%)
- Production tools: **Argo Rollouts**, **Flagger** (automated steps + metric analysis + auto-abort)

## Gotchas

- Rollback restores the Pod template only — not ConfigMaps/Secrets
- Selector must exactly match template labels or zero Pods are created
- `recreate` strategy = full downtime; only when versions can't coexist
- Undo on a broken first revision has nothing to go back to — fix the image instead
- `kubectl rollout restart` creates a new revision too (annotation change)
