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
