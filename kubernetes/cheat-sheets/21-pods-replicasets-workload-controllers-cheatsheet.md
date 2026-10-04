# Pods, ReplicaSets & Workload Controllers — Cheat Sheet

## Pods

| Concept | One-liner |
|---|---|
| Pod | Smallest deployable unit: one IP, shared network namespace, shared volumes, one lifecycle |
| Pause container | Holds the Pod's network namespace; real containers join it |
| Same Pod networking | All containers share localhost — no two can use the same port |
| Multi-container rule | One Pod when containers share lifecycle/scale together (sidecars); separate Pods otherwise |

## Init vs sidecar vs ephemeral

| Type | Runs | Use for |
|---|---|---|
| Init container | Sequentially, to completion, before app starts | Wait for DB, migrations, config fetch |
| Sidecar | Alongside app for its whole lifetime | Log shipper, proxy, metrics exporter |
| Native sidecar (1.28+) | Init container with `restartPolicy: Always` | Service mesh proxies, log agents |
| Ephemeral container | Temporarily attached via `kubectl debug` | Debugging distroless images; gone on restart |

## Probes

| Probe | Failure means | Kubelet action |
|---|---|---|
| `livenessProbe` | Container is dead/locked | Restart the container |
| `readinessProbe` | Container can't take traffic | Remove Pod from Service endpoints (no restart) |
| `startupProbe` | Container still booting | Pause liveness checks until it passes |

- Never let a slow startup trip a liveness probe — that's an infinite restart loop.
- Readiness failing ≠ restart. CrashLoopBackOff ≠ phase — it means repeated crashes; check logs + exit code.

## Pod phases

| Phase | Meaning |
|---|---|
| `Pending` | Waiting: unscheduled, pulling image, or unbound PVC — read `kubectl describe` Events |
| `Running` | At least one container running |
| `Succeeded` / `Failed` | Terminal states (Jobs) |
| `Unknown` | Node lost contact |

## Workload controllers

| Controller | Guarantees | Use for |
|---|---|---|
| ReplicaSet | N identical Pods running; adopts by label selector | Self-healing stateless replicas (usually via Deployment) |
| DaemonSet | Exactly one Pod per (selected) node | Log agents, monitoring, CNI, kube-proxy |
| StatefulSet | Stable hostname (`db-0`), ordered create/delete, per-Pod PVC | Databases, Kafka, Zookeeper |
| Job | Task runs to completion (`completions`, `parallelism`, `backoffLimit`) | Migrations, batch, imports |
| CronJob | Job on a cron schedule | Nightly backups, reports; pause with `suspend: true` |

## Rules that never change

- Pods are disposable — controllers provide durability, not the Pod.
- ReplicaSet adoption is by label matching: a manually created Pod with matching labels counts (and may get culled).
- Pod template labels must match the controller's selector; changing a live selector is forbidden.
- `emptyDir` dies with the Pod. Persist with a PVC; stable identity needs a StatefulSet.
- StatefulSets need a headless Service (`clusterIP: None`) + `volumeClaimTemplates`.
- Finished Jobs pile up in etcd — set `ttlSecondsAfterFinished`.

## Debug flow

1. `kubectl describe pod <name>` → read **Events** (the answer key)
2. `kubectl logs <pod> --previous` → what died and why
3. Check in order: resources, affinity/nodeSelector, taints, image pull, PVC binding
4. `kubectl get pods -o wide` → which node; `kubectl top nodes` → capacity

## kubectl one-liners

```bash
kubectl run two-box --image=nginx --dry-run=client -o yaml   # scaffold a pod manifest
kubectl debug -it <pod> --image=busybox --target=<container>  # ephemeral debug container
kubectl get pods -l app=web -w                               # watch reconciliation live
kubectl scale rs web-rs --replicas=5                          # scale a replicaset
kubectl rollout status deploy/web && kubectl rollout undo deploy/web
kubectl delete job hello-job                                  # finished jobs clean manually if no TTL
```
