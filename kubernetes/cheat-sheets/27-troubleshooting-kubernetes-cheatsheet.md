# Troubleshooting Kubernetes — Cheat Sheet

*Dense command reference. Keep it open in the next tab.*

## Pod states at a glance

| State | Meaning | First command |
|---|---|---|
| CrashLoopBackOff | Container keeps crashing on start | `kubectl logs <pod> --previous` |
| ImagePullBackOff | Can't pull the image | `kubectl describe pod` → Events |
| Pending | Scheduler can't place it | `kubectl describe pod` → scheduler warnings |
| OOMKilled | Exceeded memory limit (exit 137) | `kubectl describe pod` → Last State |
| Evicted | Node pressure killed it | `kubectl describe node` → Conditions |
| ContainerCreating | Stuck mounting/pulling | `kubectl describe pod` → Events |

## The core debug flow

```bash
kubectl get pods -A --sort-by=.status.containerStatuses[0].restartCount
kubectl logs <pod>                        # current container logs
kubectl logs <pod> --previous             # last crashed container — the important one
kubectl logs <pod> -c <container>         # specific container in multi-container pod
kubectl describe pod <pod>                # events, probes, exit codes, node
kubectl get events --sort-by=.lastTimestamp   # cluster-wide event stream
kubectl get events -w                      # watch events live
```

## Describe-pod decoding

```bash
kubectl describe pod <pod> | grep -A 8 "Last State"   # how the previous container died
kubectl describe pod <pod> | grep -A 12 "Events:"    # scheduler + kubelet verdicts
kubectl get pod <pod> -o yaml | grep -B2 -A5 "state:" # raw container state block
```

Exit codes worth memorizing: `1` = app error, `2` = misused shell builtin, `126` = not executable, `127` = command not found, `137` = OOMKilled (128+9 SIGKILL), `139` = segfault (128+11 SIGSEGV), `143` = SIGTERM (128+15).

## Into the pod

```bash
kubectl exec -it <pod> -- sh                          # shell in the container
kubectl exec -it <pod> -- curl -s localhost:8080/healthz   # is the app up locally?
kubectl exec -it <pod> -c <container> -- env | sort   # env vars actually set
kubectl cp <pod>:/var/log/app.log ./app.log          # copy a log file out
kubectl port-forward <pod> 8080:8080                 # bypass service/ingress
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>  # ephemeral debug toolbox
kubectl debug node/<node> -it --image=busybox        # shell on the node, no SSH
```

## Networking triage

```bash
kubectl get endpoints <svc>               # empty? -> selector bug or no ready pods
kubectl get svc <svc> -o yaml | grep -A 3 selector
kubectl run tmp --image=busybox --restart=Never -it --rm -- nslookup <svc>.<ns>
kubectl run tmp --image=busybox --restart=Never -it --rm -- wget -qO- http://<svc>.<ns>:80
kubectl get ingress <name> -o yaml        # backend service name + port + rules
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | tail -50
```

DNS FQDN reminder: `myservice` resolves only in its own namespace; cross-namespace needs `myservice.other-ns.svc.cluster.local`.

## Nodes and resources

```bash
kubectl describe node <node> | grep -B 1 -A 12 "Conditions:"  # pressure conditions
kubectl top node                          # node CPU/memory (needs metrics-server)
kubectl top pod -A --sort-by=memory       # biggest memory consumers first
kubectl get pods -A -o wide | grep <node> # everything scheduled on a bad node
kubectl drain <node> --ignore-daemonsets  # evacuate a node safely
kubectl uncordon <node>                   # allow scheduling again
```

## Rollouts — stuck or broken

```bash
kubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>
kubectl rollout undo deploy/<name>        # roll back to previous revision
kubectl rollout undo deploy/<name> --to-revision=2
kubectl rollout restart deploy/<name>     # rolling restart, keeps spec
kubectl describe deploy <name> | grep -A 5 "Conditions:"   # ProgressDeadlineExceeded?
```

## Control plane health

```bash
kubectl get componentstatuses             # older clusters
kubectl cluster-info
kubectl get --raw='/healthz'              # API server health
etcdctl endpoint health --cluster         # etcd quorum (self-hosted)
kubectl get leases -n kube-system         # leader election state
```

## Useful flags

`--previous` (last crashed logs), `-A` (all namespaces), `-o wide` (node + IP), `--sort-by`, `-w` (watch), `--field-selector=status.phase=Failed`, `--dry-run=client -o yaml` (render without applying).
