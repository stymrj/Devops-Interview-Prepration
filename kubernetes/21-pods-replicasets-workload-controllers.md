# Pods, ReplicaSets & Workload Controllers Interview Preparation Guide

*How to Answer Pods, ReplicaSets & Workload Controller Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Pods: The Atomic Unit](#pods-the-atomic-unit)
2. [Multi-Container Pod Patterns](#multi-container-pod-patterns)
3. [Pod Lifecycle and Probes](#pod-lifecycle-and-probes)
4. [ReplicaSets and Selectors](#replicasets-and-selectors)
5. [DaemonSets and StatefulSets](#daemonsets-and-statefulsets)
6. [Jobs and CronJobs](#jobs-and-cronjobs)
7. [Common Interview Traps](#common-interview-traps)

---

## Pods: The Atomic Unit

### Q1: Why is a Pod the smallest deployable unit in Kubernetes, and not a container?

**How to Answer:**

"Because containers in one app often need to behave like processes on the same machine — they share localhost, they share volumes, and they're scheduled and killed together. A Pod wraps that bundle: one IP, shared network namespace, optional shared volumes, one lifecycle. Think of the classic pattern — an app container plus a log-shipping sidecar that have to live and die together. If Kubernetes scheduled them separately, the sidecar could land on a different node and the whole pattern breaks. So the Pod exists to express 'these containers are one unit'."

**Key Point:** "Containers in a Pod share network, volumes, and fate — they're scheduled, started, and killed as one unit."

---

### Q2: Walk me through what happens when I run `kubectl run nginx --image=nginx`.

**How to Answer:**

"kubectl sends a Pod spec to the API server, which validates it and writes it to etcd with no node assigned yet. The scheduler sees an unscheduled Pod, filters nodes that can't run it — not enough CPU, taints, node selectors — scores the survivors, and binds the Pod to the best node. The kubelet on that node reads the spec, asks containerd to pull the nginx image and start the container, sets up the Pod IP and volumes, and reports the status back. The whole thing is a reconciliation loop: desired state in etcd, schedulers and kubelets continuously driving reality toward it."

**Key Point:** "API server validates and stores, scheduler binds a node, kubelet starts containers — desired state drives everything."

---

### Q3: Can containers inside a Pod talk to each other over localhost? How does Pod networking work under the hood?

**How to Answer:**

"Yes — every container in a Pod shares one network namespace, so they're all on localhost to each other and each gets the same Pod IP. Under the hood, the runtime creates a 'pause' container first that holds the network namespace, and every real container joins it. That means two containers can't listen on the same port, which is a classic interview trap. The CNI plugin then assigns the Pod IP and wires routing so every Pod can reach every other Pod without NAT — that's the flat networking model Kubernetes demands. One line to remember: same Pod means same IP, shared localhost, port conflicts possible."

**Key Point:** "All containers in a Pod share one network namespace via the pause container — same IP, localhost between them, no duplicate ports."

---

### Q4: When should I put multiple containers in one Pod versus separate Pods?

**How to Answer:**

"Same Pod when the containers are tightly coupled — they scale together, share the same lifecycle, and need to talk fast over localhost or shared files. Classic patterns: a log-shipping sidecar, a metrics exporter like the Prometheus node-exporter pattern, or a service mesh proxy like Envoy injected by Istio. Separate Pods when they scale independently, deploy at different speeds, or belong to different teams. The test I use: if you'd never want container A running without container B, they belong in one Pod. Anything else, keep them separate so you can scale and roll them independently."

**Key Point:** "One Pod when containers must live, scale, and die together; separate Pods when they have independent lifecycles."

---

## Multi-Container Pod Patterns

### Q5: What's the difference between an init container and a sidecar?

**How to Answer:**

"Init containers run to completion before the app containers start — in order, one after another — and if one fails, the whole Pod restarts from the first init container. I use them for setup: waiting for a database to be reachable, downloading config, running migrations. Sidecars are regular containers that run alongside the app for its whole lifetime — log shippers, proxies, config reloader. Since 1.28, native sidecars exist: you can set `restartPolicy: Always` on an init container so it starts before the app and keeps running, which is how service meshes inject proxies now. Rule of thumb: init for setup that must finish, sidecar for helpers that run forever."

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
```

**Key Point:** "Init containers run sequentially to completion before app start; sidecars run for the Pod's whole lifetime."

---

### Q6: What are ephemeral containers, and when would you use one?

**How to Answer:**

"I use ephemeral containers for debugging a running Pod I can't restart — my app image is distroless with no shell, so I attach a temporary debug container into the Pod's namespaces. You add one with `kubectl debug -it <pod> --image=busybox --target=<container>`, and it shares the target's process namespace so I can see its processes. The catch is they're truly ephemeral: you can't remove one once added, and they vanish when the Pod restarts. Interviewers love this question because the obvious answer — exec into the Pod — fails when the image has no shell."

**Key Point:** "Ephemeral containers attach a temporary debug shell to a running Pod — added via kubectl debug, gone on restart."

---

## Pod Lifecycle and Probes

### Q7: Explain the Pod lifecycle phases — and which one confuses people the most.

**How to Answer:**

"A Pod moves through Pending, Running, Succeeded, Failed, and Unknown. Pending means it's waiting — usually for scheduling or image pulls, so `kubectl describe` shows you why. Running means at least one container is running; Succeeded and Failed are terminal states, mostly for Jobs. The confusing one is CrashLoopBackOff — it's not a phase, it's kubelet saying 'your container keeps dying and I'm backing off before retrying'. When I see CrashLoopBackOff I check the container logs and the exit code first, because nine times out of ten it's a bad config or a missing env var, not Kubernetes."

**Key Point:** "CrashLoopBackOff isn't a phase — it means your container keeps dying; check logs and exit codes, not the cluster."

---

### Q8: What's the difference between liveness, readiness, and startup probes?

**How to Answer:**

"They answer three different questions. Liveness asks 'is this container still alive?' — if it fails, kubelet kills and restarts the container, so use it for deadlocks, never for slow startup. Readiness asks 'can this container take traffic?' — if it fails, the Pod is pulled out of Service endpoints but not restarted, which is what you want during a slow boot or dependency outage. Startup asks 'has this container finished starting?' — while it runs, liveness is paused, so a 60-second startup probe protects apps that take a while to boot from being killed by an aggressive liveness check. The classic mistake is a liveness probe that fails during slow startup and creates an infinite restart loop."

```yaml
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Key Point:** "Liveness restarts dead containers, readiness removes from endpoints without restarting, startup pauses liveness during slow boots."

---
## ReplicaSets and Selectors

### Q9: What's the difference between a ReplicaSet and a ReplicationController?

**How to Answer:**

"ReplicationController is the legacy API — it only supports equality-based selectors like `app=web`, and it's effectively deprecated. ReplicaSet is the replacement with the same job — keep N identical Pods running — but with set-based selectors like `app in (web, api)` and `tier notin (db)`. In practice I almost never create a ReplicaSet directly; Deployments create and manage them. The one place ReplicaSets show up alone is when I need the self-healing loop without rollout behavior. Interviewers ask this to check whether you know ReplicaSets are the mechanism and Deployments are the management layer on top."

```yaml
selector:
  matchLabels: { app: web }
```

**Key Point:** "ReplicationController is deprecated; ReplicaSet adds set-based selectors — and Deployments manage ReplicaSets, you rarely create one directly."

---

### Q10: How does a ReplicaSet know which Pods are "its" Pods?

**How to Answer:**

"Through label selectors, and it's purely a matching rule — not ownership by creation. The ReplicaSet's selector must match the labels on its Pod template, and any existing Pod with matching labels gets adopted by the ReplicaSet, even one you created manually with `kubectl run`. That's why the Pod template labels and the selector have to agree, and why accidentally matching an existing Pod means your replica count math breaks. If a Pod dies, the ReplicaSet controller just counts matching Pods, sees fewer than desired, and creates more. This adoption behavior is also why changing a ReplicaSet's selector on a live object is forbidden — it would orphan or steal Pods unpredictably."

**Key Point:** "ReplicaSets adopt any Pod whose labels match their selector — ownership is by label matching, not by who created the Pod."

---

## DaemonSets and StatefulSets

### Q11: When do you use a DaemonSet, and what are the classic examples?

**How to Answer:**

"DaemonSet guarantees exactly one Pod per node — or per selected node — which is what I need for node-level infrastructure. Classic examples: log agents like Fluent Bit, monitoring agents like the Datadog agent, CNI networking components, and kube-proxy itself. The use case test is simple: the workload makes sense once per machine, not per application. DaemonSets respect taints and tolerations, so I can schedule a monitoring agent onto nodes tainted for GPU workloads if the agent needs to watch those too. One gotcha: rolling updates on DaemonSets update node by node, so I watch for the surge of churn on large clusters."

**Key Point:** "DaemonSet runs one Pod per node for machine-level agents — log shippers, monitoring, CNI, kube-proxy."

---

### Q12: What's special about StatefulSets compared to Deployments?

**How to Answer:**

"StatefulSets give each Pod a stable identity: a predictable hostname like `db-0`, `db-1`, and a stable persistent volume that follows the Pod through restarts. They're created and deleted in order — `db-0` fully starts before `db-1` — which databases with leader election depend on. That comes from a headless Service plus `volumeClaimTemplates` that provision one PVC per replica. I reach for StatefulSets for databases like Postgres or Cassandra, and message systems like Kafka and Zookeeper. The trade-off is slower, more careful operations: no random scaling, and deletes remove Pods in reverse order."

```yaml
serviceName: "db"
volumeClaimTemplates:
- metadata: { name: data }
  spec: { accessModes: ["ReadWriteOnce"], resources: { requests: { storage: 10Gi } } }
```

**Key Point:** "StatefulSets give stable hostnames, ordered startup, and per-Pod volumes — that's the whole point of running databases on Kubernetes."

---

### Q13: My database Pod restarts and loses its data. What did I do wrong?

**How to Answer:**

"You used a Deployment with an `emptyDir` volume or no volume at all. Container filesystems are ephemeral — when the container dies, everything not on a mounted volume is gone, and emptyDir dies with the Pod. The fix is a PersistentVolumeClaim: for a single-instance database a Deployment with a PVC is honestly fine, but once you need replicas or ordered failover, move to a StatefulSet. This is a classic 'say the right fix, not the fancy one' question: the interviewer wants to hear 'mount a PVC' first, and StatefulSet only if there's a real reason. I also check the storage class, because a PVC in Pending usually means dynamic provisioning has no matching provisioner."

**Key Point:** "Container filesystems are ephemeral — persist with a PVC; reach for StatefulSets when replicas need stable identity, not by default."

---

## Jobs and CronJobs

### Q14: When do you use a Job instead of a Deployment?

**How to Answer:**

"A Deployment wants N Pods running forever; a Job wants a task to run to completion. Database migrations, one-off data imports, batch processing, report generation — anything with a defined end. The Job controller creates Pods and retries them on failure up to `backoffLimit`, and `completions` plus `parallelism` control how many successful runs I need and how many run at once. For scheduling, CronJob wraps Job with a cron expression. The production details I always mention: set `activeDeadlineSeconds` so a hung job can't run forever, and set `ttlSecondsAfterFinished` so finished Pods get cleaned up instead of piling up in etcd."

```yaml
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 3
  ttlSecondsAfterFinished: 3600
  template: { ... }
```

**Key Point:** "Deployments run forever, Jobs run to completion — use Jobs for migrations and batch work, with backoffLimit and TTL cleanup."

---

## Common Interview Traps

### Q15: A Pod is stuck in Pending. Walk me through your debugging order.

**How to Answer:**

"I run `kubectl describe pod` first and read the Events section — it tells me the actual reason. The usual suspects in order: no node has enough CPU or memory, which shows as 'Insufficient cpu' and I check `kubectl top nodes` and resource requests; a node selector or affinity that matches nothing; taints without tolerations; an image that can't be pulled, which shows as ImagePullBackOff instead of plain Pending; or a PVC that can't bind because the storage class has no provisioner. I never guess — the Events section is the answer key, and I say that in interviews. One more: on a fresh cluster, Pending with 'no nodes available to schedule' usually means the nodes aren't Ready yet."

**Key Point:** "kubectl describe pod, read Events, fix what it names — resources, affinity, taints, image pulls, unbound PVCs, in that order."

---

*That's the whole topic — Pods as the unit, ReplicaSets for self-healing, and the right controller for each workload shape. Next: Deployments & rollout strategies.*
