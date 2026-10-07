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
