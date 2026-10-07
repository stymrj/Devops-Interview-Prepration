# Troubleshooting Kubernetes Interview Preparation Guide

*How to Answer Troubleshooting Kubernetes Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Pod Troubleshooting Workflow](#pod-troubleshooting-workflow)
2. [Networking and Service Debugging](#networking-and-service-debugging)
3. [Node and Resource Problems](#node-and-resource-problems)
4. [Incident Triage Methodology](#incident-triage-methodology)

---

## Pod Troubleshooting Workflow

### Q1: A pod is stuck in CrashLoopBackOff. Where do you start?

**How to Answer:**

"I always start with logs before anything else. `kubectl logs <pod> --previous` shows me the last crashed container's output — that's where the actual error usually lives. If there's nothing there, I run `kubectl describe pod` and read the Events section, which shows the restart pattern and backoff timings."

"Most of the time it's one of three things: the app crashes on startup (bad config, missing env var), a failed liveness probe killing a healthy app, or an OOM kill. The describe output tells me which — exit code 137 means OOM, probe failures show up in events."

"My workflow is: logs first, describe second, fix the root cause rather than restarting the pod into the same crash loop again."

**Key Point:** "Logs from the previous container first, describe pod second — the exit code tells you the category of failure."

---

### Q2: How do you debug a pod stuck in ImagePullBackOff?

**How to Answer:**

"ImagePullBackOff is almost always authentication or a wrong image reference. I check the exact image name and tag first — a typo in the tag or a tag that doesn't exist is the most common cause. Then I look at the describe events for the specific pull error."

"If the image is in a private registry, I verify the imagePullSecrets actually exists and is referenced correctly in the pod spec. I've seen it where the secret exists but in the wrong namespace, or the secret has a stale password."

"One more check: network egress. If the cluster can't reach the registry at all — firewall rules, no egress — the pull fails regardless of credentials. Describe pod tells you which of these it is."

```bash
kubectl describe pod myapp-abc123 | grep -A 5 "Events"
```

**Key Point:** "It's the image name/tag, the pull secret, or registry reachability — describe pod tells you which."

---

### Q3: A pod is stuck in Pending forever. What do you look at?

**How to Answer:**

"Pending means the scheduler couldn't place the pod, so I go straight to describe and read the scheduler's warnings in Events. The usual suspects are resource requests that no node can satisfy, or a node selector/affinity rule that matches nothing."

"I also check taints and tolerations — a pod without the right toleration just sits there while the nodes all carry taints. And if the cluster has a lot of pending pods, I check whether the node pool is simply full and needs scaling."

"The fix depends on the cause: lower the requests, fix the affinity, add tolerations, or scale the node group. But the diagnosis is always the same command — the scheduler tells you exactly why it said no."

**Key Point:** "Pending is the scheduler refusing — read its reason in Events, then fix the mismatch."

---

### Q4: Your pod is Running but the app isn't responding. What's your move?

**How to Answer:**

"Running just means the container started — it says nothing about whether the app works. I exec into the pod first and check if the app process is actually up and listening. `curl localhost:<port>` from inside the pod tells me immediately whether it's an app problem or a networking problem."

"If the app answers locally but not from outside, the issue is upstream: service selectors, endpoints, or NetworkPolicies. I check `kubectl get endpoints` to make sure the service actually has backend addresses — a wrong selector label silently gives you an empty endpoints list."

"If the app doesn't answer locally either, I look at the app's own logs and readiness probe. A readiness probe misconfiguration can keep a healthy pod out of rotation, and the app might genuinely be hung."

```bash
kubectl exec -it myapp-abc123 -- curl -s localhost:8080/healthz
```

**Key Point:** "Test from inside the pod first — local curl separates app bugs from networking bugs instantly."

---

## Networking and Service Debugging

### Q5: Pods can't resolve service DNS names. How do you troubleshoot?

**How to Answer:**

"DNS in Kubernetes goes through CoreDNS, so that's where I look. First I test resolution directly from a pod — `nslookup myservice` or `dig` — and see whether it fails everywhere or just from some pods. If it's everywhere, CoreDNS itself is probably down or misconfigured."

"Then I check the CoreDNS pods are running and read their logs. A broken Corefile or a failed upstream resolver shows up there. I also check the pod's `/etc/resolv.conf` — the nameserver should point at the cluster DNS service, and a broken kube-dns service IP explains a lot."

"The classic gotcha is pods in a different namespace forgetting the full FQDN — `myservice` resolves only within its own namespace, so cross-namespace calls need `myservice.other-ns.svc.cluster.local`. That one wastes hours if you don't spot it."

**Key Point:** "Test from the pod, then check CoreDNS health — and remember DNS names are namespace-scoped by default."

---

### Q6: Your Ingress is returning 502s. Walk me through the debug path.

**How to Answer:**

"A 502 from an ingress usually means the ingress controller reached the backend but got a bad response — or couldn't reach it at all. I start at the backend: check the service endpoints have addresses, and that the pods are actually passing their readiness probes. An ingress with a backend that has no ready endpoints returns 502."

"Then I check the ingress resource itself — the backend service name, port, and the path rules. A port mismatch between the ingress backend config and the service's targetPort is a classic. `kubectl describe ingress` plus the controller's logs usually surface it."

"Finally I look at the ingress controller logs and check whether the controller is even running healthy — I've seen 502s where the controller's own pods were crash-looping and nobody noticed because the ingress object looked fine."

```bash
kubectl get endpoints web-svc
```

**Key Point:** "502s are backend reachability — verify endpoints exist, probes pass, and the ingress port mapping matches."
---

## Node and Resource Problems

### Q7: A pod keeps getting OOMKilled. What's your process?

**How to Answer:**

"Exit code 137 with reason OOMKilled means the container exceeded its memory limit — the kernel killed it, not the app. First I confirm with `kubectl describe pod` and look at the last state. Then I check whether the limit is actually realistic or whether the app has a memory leak."

"The quick fix is raising the memory limit, but I always ask why it died first. If memory grows linearly until the kill, that's a leak and a bigger limit just delays it. If it dies on traffic spikes, the limit was too tight for the workload."

"Long-term, I set proper requests and limits — requests for scheduling, limits for protection — and alert on memory usage approaching the limit so we catch it before the kill. VerticalPodAutoscaler recommendations help size this over time."

**Key Point:** "137 means the kernel killed you — distinguish a leak from an undersized limit before you raise it."

---

### Q8: Pods are getting evicted and nodes show disk or memory pressure. What do you do?

**How to Answer:**

"Node pressure eviction is kubelet protecting the node — when disk or memory crosses thresholds, it starts killing pods, lowest priority and QoS first. `kubectl describe node` shows the pressure conditions clearly, so I confirm which resource is the problem."

"For disk pressure, the usual culprits are runaway logs filling the disk or old container images. I clean up unused images and check whether log rotation is configured. For memory pressure, I look for pods without limits — a pod with no memory limit can eat everything and trigger eviction of its neighbors."

"Best-effort pods die first in evictions, so setting requests and limits isn't optional in production. After stabilizing, I add node-level alerting on pressure conditions and make sure the node has enough headroom."

```bash
kubectl describe node worker-1 | grep -A 3 "Conditions:"
```

**Key Point:** "Eviction is kubelet defending the node — find which resource is pressured, then give every pod proper requests and limits."

---

### Q9: The whole cluster is misbehaving — how do you check control plane health?

**How to Answer:**

"I work top-down: API server first, then etcd, then scheduler and controller-manager. `kubectl get componentstatuses` is the quick check on older clusters, but on modern managed clusters I look at the control plane metrics and logs instead — you don't always have direct access."

"On self-hosted clusters I check etcd health directly — etcd quorum loss is the scariest cluster failure, and it usually comes from disk latency or a full disk on the etcd members. `etcdctl endpoint health` tells the story fast."

"For managed clusters like EKS or GKE, control plane issues are usually exposed through the cloud provider's status and the API server's request latency metrics. I also keep an eye on leader election — a scheduler that lost its lease just stops scheduling while everything else looks fine."

**Key Point:** "Top-down: API server, then etcd quorum, then scheduler and controller-manager — etcd disk issues are the classic silent killer."

---

### Q10: A deployment is stuck mid-rollout. How do you recover?

**How to Answer:**

"`kubectl rollout status` shows where it stopped, and `kubectl describe deployment` tells me why — usually the new ReplicaSet's pods never became ready. Bad image tag, failed probes, or a config error in the new version are the usual causes."

"First decision: roll forward or roll back. If I know the fix and it's small, I fix and push a new revision. If I'm unsure, I roll back — `kubectl rollout undo` restores the last good ReplicaSet in seconds, and arguing with a broken rollout during an incident is how outages get longer."

"To prevent repeats, I set `progressDeadlineSeconds` so a stuck rollout fails loudly instead of hanging forever, and I always test the new image's health endpoint in a lower environment before promoting it."

**Key Point:** "When in doubt, roll back first and diagnose later — a working old version beats a debated new one."

---

## Incident Triage Methodology

### Q11: Production is down and everyone's looking at you. What's your triage process?

**How to Answer:**

"I go wide to narrow. First: what's the blast radius — which services, which namespaces, is it one pod or the whole cluster? Then I check the recent change — in my experience most incidents follow a deploy, a config change, or an infra change within the last hour."

"Then I layer: pod health first (`kubectl get pods -A` sorted by restarts), then nodes, then ingress and DNS, then external dependencies. I fix or mitigate before I fully understand — scale up, roll back, drain a bad node — and do the deep root-cause after the bleeding stops."

"Throughout, I keep one person on comms and one timeline. The biggest incident mistakes I've seen are five people running the same kubectl command and nobody writing down what changed when."

**Key Point:** "Blast radius, then recent changes, then layer-by-layer — mitigate first, root-cause second, and keep a timeline."

---

### Q12: What tools do you keep ready for deep debugging in Kubernetes?

**How to Answer:**

"kubectl alone covers most of it — logs, describe, exec, port-forward, and rollout commands are my daily drivers. For pods I can't exec into because they're distroless or crashed, I use `kubectl debug` with an ephemeral container to attach a debugging toolbox to the running pod."

"I keep a debug image handy — something like netshoot — for network issues: it has dig, curl, tcpdump, everything the stripped-down app image lacks. For node-level problems I use `kubectl debug node/` to get a shell on the node without SSH."

"Beyond kubectl, I rely on the observability stack we covered in earlier guides — Prometheus metrics to see what changed, and centralized logs to correlate across pods. The interview point is: debugging tools find the symptom, but metrics and logs find the cause."

```bash
kubectl debug -it myapp-abc123 --image=nicolaka/netshoot --target=myapp
```

**Key Point:** "Ephemeral debug containers plus a netshoot-style toolbox — you can debug any pod without rebuilding its image."
