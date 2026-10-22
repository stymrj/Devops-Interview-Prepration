# Live Outage Triage — Cheat Sheet

## First five minutes
```
scope: whole app / one endpoint / one region / one tenant?
last deploy time → most outages correlate
error graph + infra graphs side by side
declare incident, open timeline, pull service owner
```

## 504 gateway timeouts
```
# ALB 504 signature: elb_status_code=504, target_status_code=-, target_processing_time ~ idle timeout (60s)
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime --dimensions Name=LoadBalancer,Value=<lb-arn>
# healthy-but-slow targets = app problem | flapping targets = infra/config | high request count = capacity
```

## p99 latency spike
```
diff the deploy → check slowest downstream (DB slow queries, cache hit rate, pool saturation)
psql: SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
can't isolate fast → roll back, investigate in staging
```

## OOMKills
```
signature: exitCode 137 + lastState.reason OOMKilled
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl top pod -l app=<name> --sort-by=memory
steady climb = leak/code fix | sudden spike = undersized limit/manifest fix
requests = steady state + headroom | limits = worst observed spike (never equal to requests)
JVM: size for total RSS, not just heap
alert before the kernel acts: memory usage approaching limit
```

## Rollback commands
```
kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>
halt rollout first → roll back second → investigate third
rollback when deploy correlates; fix forward when rollback is riskier (migrations already ran)
```

## DNS storms
```
signature: query volume + latency spike together, infra graphs otherwise fine
kubectl top pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns | grep -i "servfail\|timeout\|refused"
causes: retry loops on NXDOMAIN, no client-side caching, conntrack exhaustion, scale-event boot storms
fix: client caching, jittered retries with backoff, raise CoreDNS replicas, cache plugin TTL
```

## Pod DNS debug
```
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup <service>        # reproduces from inside
kubectl exec -it <pod> -- getent hosts <service>    # tests alternate resolver path
sysctl net.netfilter.nf_conntrack_count && sysctl net.netfilter.nf_conntrack_max
```

## DiskPressure
```
kubectl describe node <node> | grep -A3 Pressure
du -sh /var/lib/docker /var/log /var/lib/kubelet
docker image prune -a --filter "until=168h"
journalctl --vacuum-time=7d
alert at 70% disk, not at eviction time
```

## Connection pool exhaustion
```
psql: SELECT count(*), state FROM pg_stat_activity GROUP BY state;
psql: SELECT pid, now()-query_start AS idle_for, query FROM pg_stat_activity
      WHERE state='idle in transaction' ORDER BY query_start LIMIT 10;
fix: SELECT pg_terminate_backend(<pid>) for worst offenders
prevent: statement_timeout + idle_in_transaction_session_timeout
```

## Cache stampede
```
signature: cache hit rate → ~0, DB CPU spikes in lockstep
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
fix now: warm cache, request coalescing (single rebuild per key)
prevent: jittered TTLs, stale-while-revalidate
```

## Incident comms one-liners
```
"Impact: <who> can't <what> since <time>."
"Current theory: <X>. Next step: <Y>, ETA <N> min."
"Decision: rolling back to <version> — it's the fastest reversible move."
"Timeline updated — next update in 15 min."
```

---
*Companion to the [Live Outage Triage Scenarios guide](57-live-outage-triage-scenarios.md).*
