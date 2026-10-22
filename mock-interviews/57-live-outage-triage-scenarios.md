# Live Outage Triage Scenarios Interview Preparation Guide

*How to Answer Outage Triage Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [The Triage Playbook](#the-triage-playbook)
2. [504s and Upstream Timeouts](#504s-and-upstream-timeouts)
3. [OOMKills and Memory Pressure](#oomkills-and-memory-pressure)
4. [DNS Storms](#dns-storms)
5. [More Live Fire Scenarios](#more-live-fire-scenarios)

---

## The Triage Playbook

### Q1: Walk me through your first five minutes when a "site is down" alert fires.

**How to Answer:**

"I don't touch anything yet — I confirm scope first. I check whether it's the whole app or one endpoint, one region, one tenant. I look at the last deploy time, because most outages I have seen land within minutes of a release. Then I open the error graph and the infrastructure graphs side by side. And I declare incident, pull in whoever owns the service, and start a shared timeline."

**Key Point:** "Scope first, blame the deploy second, start the timeline immediately."

---

### Q2: How do you decide between rolling back and fixing forward during an outage?

**How to Answer:**

"I roll back when I have a deploy correlated to the incident and a known-good version to return to. Rollbacks are boring and reversible, which is exactly what I want at 2 AM. I fix forward when the rollback is riskier than the fix — like a schema migration that already ran. I say the tradeoff out loud in the call so the decision is shared. And either way, I note the decision in the timeline."

**Key Point:** "Default to rollback — it's the fastest reversible move you have."

---

## 504s and Upstream Timeouts

### Q3: An ALB starts returning 504s. How do you figure out whether it's the app or the infrastructure?

**How to Answer:**

"A 504 means the load balancer waited and got nothing back, so the problem is upstream of it. I check target response time first — if it spiked, the app is slow or hung. Then I check target health: healthy targets that answer slowly point at the app; targets flapping unhealthy point at infra or bad config. I also compare request count against capacity — sometimes it is just too much traffic for too few instances."

```bash
# check target health and latency around the incident window
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime --dimensions Name=LoadBalancer,Value=<lb-arn>
```

**Key Point:** "504s are always upstream — health checks tell you if it's slow code or dead targets."

---

### Q4: Your app's p99 latency tripled after a deploy but no single error stands out. What do you do?

**How to Answer:**

"I treat it like a slow leak, not a crash. I diff the deploy first — new dependency, new query, a config change someone slipped in. Then I check downstream: database slow queries, a cache hit-rate drop, a saturated connection pool. I look at the dependency graph because p99 spikes usually come from one slow call on the hot path. And if I can't isolate it fast, I roll back and investigate in staging."

```bash
# find the slow queries introduced around deploy time
psql -c "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"
```

**Key Point:** "p99 spikes live on the hot path — diff the deploy, then check the slowest dependency."

---

## OOMKills and Memory Pressure

### Q5: A pod keeps getting OOMKilled. How do you debug it?

**How to Answer:**

"I confirm it's actually OOM first — exit code 137 and a lastState terminated reason of OOMKilled in the pod status. Then I watch the memory graph during traffic to see if it grows steadily or spikes suddenly. A steady climb is a leak or unbounded cache; sudden spikes are usually a traffic burst or a bad payload. I check whether limits are realistic versus actual usage, and I add memory profiling before I touch the limit."

```bash
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl top pod -l app=<name> --sort-by=memory
```

**Key Point:** "137 plus OOMKilled in lastState — then decide if it's a leak or an undersized limit."

---

### Q6: How do you size memory requests and limits so OOMKills stop happening?

**How to Answer:**

"I base requests on the steady-state usage plus headroom, and limits on the worst spike I have actually observed. The mistake I see most is setting the limit to the request — then any spike kills the pod. For JVM apps I leave room for the heap outside the heap, because containers get killed on total RSS, not just heap. And I set up an alert on memory usage approaching the limit so I hear about it before the kernel does."

```yaml
resources:
  requests:
    memory: "512Mi"   # steady-state usage + headroom
  limits:
    memory: "1Gi"     # worst observed spike, not the request value
```

**Key Point:** "Requests cover steady state, limits cover real spikes — never set them equal."

---

## DNS Storms

### Q7: What is a DNS storm, and how would you spot one?

**How to Answer:**

"A DNS storm is when something generates a flood of DNS queries — a misconfigured app retrying NXDOMAIN lookups, or every pod hammering CoreDNS because someone disabled caching. I spot it by watching DNS query volume and latency spike together while everything else looks fine. Apps show up as intermittent connection failures because name resolution is the thing flapping. In Kubernetes I check CoreDNS CPU and the cache hit rate — that's usually where it shows first."

**Key Point:** "Intermittent failures with fine infra graphs usually mean DNS is the flapping layer."

---

### Q8: Services in your cluster get intermittent "no such host" errors. Walk me through the debug.

**How to Answer:**

"I run nslookup from inside a failing pod first to reproduce it directly. If resolution fails intermittently, I check CoreDNS logs and metrics for timeouts and dropped queries. Then I look at what changed — a new service with a bad selector, an app that resolves on every request with no cache, or connection tracking table exhaustion on the node. I also check whether it correlates with scale events, because new pods resolving at boot can overwhelm DNS."

```bash
kubectl exec -it <pod> -- nslookup <service-name>
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "timeout\|refused"
```

**Key Point:** "Reproduce from inside the pod, then check CoreDNS metrics and node conntrack pressure."

---

## More Live Fire Scenarios

### Q9: A node reports DiskPressure and starts evicting pods. What happened and what do you do?

**How to Answer:**

"DiskPressure means the node's disk crossed the eviction threshold — usually logs or container images eating everything. I check du on the kubelet directories and find the fattest writers first. Short term, I clean up old images and rotate logs to stop the bleeding. Long term, I add log rotation limits and disk alerts so the node tells me at 70 percent, not at eviction time."

```bash
du -sh /var/lib/docker /var/log /var/lib/kubelet 2>/dev/null
docker image prune -a --filter "until=168h"
```

**Key Point:** "Logs and old images cause most DiskPressure — rotate both before the node evicts."

---

### Q10: You deploy at 2 PM and the error rate spikes immediately. Talk me through it.

**How to Answer:**

"I stop the rollout the second the canary metrics cross the error budget — no investigating while broken code spreads. I check whether the new version is actually serving the errors or whether it's a config or migration the deploy needed. If it's the code, I roll back to the previous revision right away. Then I pull the logs, reproduce in staging, and find the root cause with zero time pressure."

```bash
kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>
```

**Key Point:** "Halt the rollout first, roll back second, investigate third — never in another order."

---

### Q11: Your database connection pool is exhausted and the app is timing out. What now?

**How to Answer:**

"I check whether the pool is exhausted because of real traffic or leaked connections. pg_stat_activity sorted by state shows me idle-in-transaction sessions, which are the usual leak. I kill the worst offenders to restore service, then I set statement timeouts and idle-in-transaction timeouts so it can't happen again. And I look at why connections grew — usually a new deploy that opened a pool per replica instead of sharing one."

```bash
-- find the leaked/idle sessions holding connections
SELECT pid, state, query_start, query FROM pg_stat_activity
WHERE state = 'idle in transaction' ORDER BY query_start LIMIT 10;
```

**Key Point:** "Idle-in-transaction sessions are the classic leak — kill them, then add timeouts."

---

### Q12: Traffic just doubled after a marketing push and the cache hit rate collapsed. What is happening?

**How to Answer:**

"That's a cache stampede — the cache got flushed or expired under load, and every request now hits the database at once. I confirm it with the cache hit-rate graph dropping to near zero while DB CPU spikes. The quick fix is to warm the cache and add request coalescing so only one request rebuilds each key. Long term, I use jittered TTLs and stale-while-revalidate so keys never all expire at the same moment."

**Key Point:** "Hit rate near zero plus DB spike equals stampede — coalesce rebuilds and jitter your TTLs."

---

*Day 57 of 58 — one topic left: DevOps system design round.*
