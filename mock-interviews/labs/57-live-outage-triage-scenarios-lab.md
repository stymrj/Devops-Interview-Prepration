# Lab: Live Outage Triage Scenarios

Simulate the classics — 504s, OOMKills, DNS storms — in a safe local or staging cluster, and practice the exact commands you'd run at 2 AM. Each exercise: goal → commands → expected output → why it matters.

## 1. Reproduce a 504 with a slow upstream

**Goal:** See a 504 end-to-end so you recognize the signature.
**Commands:**
```bash
# run an app that sleeps longer than the LB idle timeout
docker run -d -p 8080:8080 --name slow-app python:3-slim \
  python -c "import http.server,time
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self): time.sleep(120); self.send_response(200)
http.server.HTTPServer(('0.0.0.0',8080),H).serve_forever()"
curl -m 10 http://localhost:8080   # curl gives up, LB would log a 504
```
**Expected:** `curl: (28) Operation timed out` after 10s — the backend never answered in time.
**Why it matters:** 504s are always "the upstream didn't answer in time" — never blame the LB itself.

## 2. Read ALB access logs for 504s

**Goal:** Find 504s and their upstream latency in real logs.
**Commands:**
```bash
aws s3 sync s3://<your-alb-log-bucket>/AWSLogs/ ./alb-logs/
zgrep " 504 " alb-logs/* | head -5 | awk '{print $13, $15, $16}'
# fields: elb_status_code, target_status_code, target_processing_time
```
**Expected:** Lines where `elb_status_code=504`, `target_status_code=-` and `target_processing_time` near the idle timeout (60s default).
**Why it matters:** A `-` target status code proves the backend never responded — the LB is exonerating itself.

## 3. Trigger an OOMKill in Kubernetes

**Goal:** Watch exit code 137 happen on purpose.
**Commands:**
```bash
kubectl run oom-test --image=polinux/stress --restart=Never \
  --limits=memory=64Mi -- stress --vm 1 --vm-bytes 128M --timeout 30s
sleep 35; kubectl get pod oom-test -o jsonpath='{.status.containerStatuses[0].state.terminated}'
```
**Expected:** `{"exitCode":137,"reason":"OOMKilled"}` — the kernel killed it for crossing 64Mi.
**Why it matters:** Knowing the exact signature (137 + OOMKilled) saves you from chasing app bugs that are really resource limits.

## 4. Profile pod memory during a spike

**Goal:** Practice distinguishing a leak from a traffic spike.
**Commands:**
```bash
kubectl top pod -l app=<name> --sort-by=memory
# watch it live while you generate load
watch -n 2 "kubectl top pod -l app=<name>"
```
**Expected:** Steady climb over minutes = leak or unbounded cache; sharp spike during load test = undersized limit.
**Why it matters:** Leaks get fixed in code; undersized limits get fixed in the manifest — mixing them up wastes an incident.

## 5. Simulate a DNS storm against CoreDNS

**Goal:** See what query floods look like from inside the cluster.
**Commands:**
```bash
kubectl exec -it dns-test -- sh -c \
  'for i in $(seq 1 5000); do nslookup nonexistent-$i.svc.cluster.local >/dev/null 2>&1 & done; wait'
kubectl top pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20 | grep -i servfail
```
**Expected:** CoreDNS CPU spikes, SERVFAIL/timeout lines in logs — exactly what a misconfigured retry loop does in production.
**Why it matters:** This is how you recognize a DNS storm before the app teams blame the network.

## 6. Debug "no such host" from a failing pod

**Goal:** Reproduce and isolate intermittent DNS failures.
**Commands:**
```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default
kubectl exec -it <pod> -- getent hosts <service-name>
```
**Expected:** `resolv.conf` points at the cluster DNS IP; if nslookup flaps while getent works (or vice versa), you know which resolver path is broken.
**Why it matters:** Pod DNS has multiple layers (ndots, search domains, conntrack) — you bisect them one command at a time.

## 7. Fill a disk to trigger DiskPressure (safely)

**Goal:** Watch kubelet react to disk exhaustion in staging.
**Commands:**
```bash
# on a throwaway staging node only
dd if=/dev/zero of=/var/tmp/fill-up bs=1M count=5000
kubectl describe node <node> | grep -A3 Pressure
rm /var/tmp/fill-up
```
**Expected:** `DiskPressure: True` with evictions starting; it clears minutes after you remove the file.
**Why it matters:** You learn the eviction order and the threshold numbers before a real log-rotation bug teaches them to you.

## 8. Practice a bad-deploy rollback under time pressure

**Goal:** Build muscle memory for the halt → rollback → investigate order.
**Commands:**
```bash
kubectl set image deployment/<name> <container>=broken-image:v2
kubectl rollout status deployment/<name>   # watch it hang
time kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name> && kubectl get pods -l app=<name>
```
**Expected:** Undo completes in seconds-to-a-minute; new pods run the previous image and go Ready.
**Why it matters:** In a real incident you want `rollout undo` to be reflex, not something you look up.

## 9. Find leaked DB connections

**Goal:** Spot the idle-in-transaction sessions that exhaust pools.
**Commands:**
```bash
psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
psql -c "SELECT pid, now() - query_start AS idle_for, query FROM pg_stat_activity
         WHERE state = 'idle in transaction' ORDER BY query_start LIMIT 5;"
```
**Expected:** A pile of `idle in transaction` rows with old `query_start` times — connections held open doing nothing.
**Why it matters:** This single query ends half of all "database is slow" incidents.

## 10. Simulate a cache stampede

**Goal:** Watch hit rate collapse and DB CPU spike together.
**Commands:**
```bash
# flush the cache while load is running (staging only)
redis-cli FLUSHALL
# in another terminal, watch the two graphs move
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
```
**Expected:** `keyspace_misses` jumps 10-100x while the database gets hammered — then recovers as keys rebuild.
**Why it matters:** You see the two metrics move in lockstep, so in production you'll diagnose it in seconds.

---

*Pairs with the [Live Outage Triage Scenarios guide](57-live-outage-triage-scenarios.md). Day 57 of 58.*
