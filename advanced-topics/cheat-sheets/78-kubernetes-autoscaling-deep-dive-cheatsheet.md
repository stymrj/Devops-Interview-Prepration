# Kubernetes Autoscaling Deep Dive — Cheat Sheet

## The four autoscalers

| Scaler | Scales | Watches | Acts every |
|---|---|---|---|
| HPA | Pod count | CPU / memory / custom metrics | 15s (default) |
| VPA | Pod size (requests) | Actual usage over days | On eviction / pod creation |
| Cluster Autoscaler | Node count | Unschedulable pods, idle nodes | ~10s scan loop |
| KEDA | Pod count on events | Queue length, Kafka lag, cron | Own polling interval |

HPA scales pod count, VPA scales pod size, Cluster Autoscaler scales node count, KEDA scales pod count on events — different layers, not alternatives.

## The metrics pipeline

| Piece | Job |
|---|---|
| kubelet | Reports per-pod CPU/memory |
| metrics-server | Serves the metrics API the HPA reads for resource metrics |
| Prometheus Adapter | Exposes Prometheus queries as custom metrics for the HPA |
| HPA controller | Reads the metric, computes replicas, patches the workload |

Metrics flow from kubelet to metrics-server to the HPA; custom metrics come from Prometheus via an adapter. If `kubectl top` fails, the HPA is blind.

## The HPA formula

| Piece | Meaning |
|---|---|
| Desired replicas | Current replicas × current metric ÷ target |
| Check interval | 15 seconds (default) |
| Scale-down stabilization | 5 minutes (default) — one quiet minute doesn't kill pods |
| minReplicas / maxReplicas | Hard floor and ceiling — the HPA never goes outside them |

Desired equals current times current over target — checked every 15 seconds, scale-down held back 5 minutes.

## What to scale on

| Metric | Verdict |
|---|---|
| CPU utilisation | Safe default — CPU is compressible, it just runs slower |
| Memory utilisation | Usually a trap — memory doesn't shed load, a leak just spreads |
| Requests per second | Best for user-facing services — needs the Prometheus Adapter |
| Queue length / lag | Best for workers — KEDA's home turf |
| Latency | Tempting but noisy — scale on the cause (rate, queue), not the symptom |

CPU by default, never memory alone, and graduate to request rate or queue depth when the service is latency-sensitive.

## Stopping the flap

| Fix | How |
|---|---|
| Stabilization window | Hold scale-downs 5 min, scale-ups 60s — spikes don't stampede |
| Scale policy | Cap growth: at most double replicas or +4 pods per minute |
| Sane target | 60–70% CPU, not 90% — headroom absorbs the wobble |

Flapping means the target is too tight or the cool-downs too short — widen the windows, cap the rate, target 60–70%.

## HPA at maxReplicas

| Step | What to do |
|---|---|
| Set the ceiling | From the load test — the highest count the DB and downstreams survive |
| Alert on it | "HPA at max replicas for 5 minutes" pages — the system is saturated |
| Check the nodes | Extra pods are useless if the Cluster Autoscaler can't add nodes fast enough |

HPA at maxReplicas just stops — set the ceiling from the load test, alert when it's hit, and make sure nodes can appear.

## VPA modes

| Mode | Behaviour |
|---|---|
| Off | Recommendations only — nothing changes |
| Initial | Sets requests when a pod is created |
| Auto | Evicts and recreates pods with new requests |

VPA rightsizes pod requests and restarts pods to apply them. It fights the HPA on the same metric — split the signals (VPA on memory, HPA on CPU) or run VPA in Off mode.

## VPA guardrails

| Rule | Why |
|---|---|
| A week of real traffic | A recommendation after one quiet weekend is worthless |
| minAllowed / maxAllowed | A bug can't recommend 16 CPUs or 64Mi for a Java app |
| Auto only for stateless workers | An eviction mid-request is a dropped request |
| Everything else by PR | Off mode + human review for stateful and latency-critical workloads |

A week of real traffic, min/max fences, Auto only for stateless workers.

## KEDA

| Piece | Meaning |
|---|---|
| ScaledObject | Names the workload, min/max replicas, and triggers |
| Trigger | The event source — SQS queue, Kafka lag, Redis list, cron |
| Scale to zero | When the trigger is empty, replicas go to 0 — the plain HPA can't do this |
| Cold start | First message after zero waits for a pod to boot |

KEDA scales on events and scales to zero — perfect for workers, wrong for user-facing APIs because of cold start.

## Cluster Autoscaler

| Decision | Trigger |
|---|---|
| Add a node | A pod is unschedulable and would fit on a bigger node group |
| Remove a node | Every pod on it can move elsewhere — underutilised, no local-storage pin, PDB allows eviction |
| Blocked scale-down | One unmovable pod (local volume, single replica, no PDB) holds the whole node |

Adds nodes for unschedulable pods, removes them only when every pod can move — one unmovable pod blocks the whole node.

## Cluster Autoscaler vs Karpenter

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Node selection | Fixed, predefined node groups | Provisions the exact instance a pod needs, right then |
| Speed | Slow — grows a whole group | Fast — one right-sized node |
| Consolidation | Manual | Automatic — underused nodes get replaced |
| Cost | Simple, predictable | Cheaper for varied workloads, but another controller to run |

Cluster Autoscaler for simple, predictable workloads — Karpenter when pods vary wildly in size and you want the cheapest node that fits, right now.

## Sale-day checklist

| Layer | Check |
|---|---|
| HPA | Request rate metric, 60% CPU backstop, min 3, max from the load test, 5-min scale-down window |
| Nodes | Pre-warmed an hour before — nodes take minutes to appear |
| Database | Connection pool sized for max replicas |
| Downstreams | Payment gateway rate limit confirmed in writing |
| Guardrails | PodDisruptionBudget so scale-down never takes the last healthy pod |
| Alert | HPA at maxReplicas for 5 minutes pages immediately |

HPA on request rate, nodes pre-warmed before the sale, pools and rate limits sized for the max — and alert at the ceiling.
