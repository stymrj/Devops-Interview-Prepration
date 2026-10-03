# Kubernetes Architecture — Cheat Sheet

## Control plane components

| Component | Does | Runs as |
|---|---|---|
| `kube-apiserver` | Only front door; validates + auth; only writer to etcd | static pod |
| `etcd` | Holds ALL cluster state (key-value) | static pod |
| `kube-scheduler` | Binds unscheduled pods to nodes (filter → score) | static pod |
| `kube-controller-manager` | Control loops: node, replication, endpoints | static pod |
| `cloud-controller-manager` | Cloud API loops: nodes, LB, routes | static pod / deployment |

## Worker node components

| Component | Does |
|---|---|
| `kubelet` | Node agent: registers node, runs assigned pods, reports status |
| container runtime (`containerd`) | Pulls images, runs containers via CRI |
| `kube-proxy` | Programs iptables/IPVS rules for Services |

## Rules that never change

- Everything talks **through the API server** — components never talk to each other directly.
- Only the API server writes to etcd; every other control plane component is **stateless**.
- **Scheduler assigns, kubelet executes.** Kubelet never schedules.
- etcd needs **odd quorum** (3 or 5) — never scale it up for "performance".

## The apply flow (kubectl apply → running)

1. API server validates + writes object to etcd
2. Deployment controller → creates ReplicaSet
3. ReplicaSet controller → creates Pod objects (unscheduled)
4. Scheduler → filters nodes → scores → writes binding
5. Kubelet on node → pulls image → starts containers
6. Endpoints controller → updates Service endpoints

## Scheduling

```bash
kubectl describe pod <pod> | grep -A5 Events   # see scheduler's binding
kubectl taint nodes <node> key=val:NoSchedule  # keep pods off a node
kubectl taint nodes <node> key=val:NoSchedule-  # remove taint
# scheduling knobs: nodeSelector, nodeAffinity, podAffinity,
# podAntiAffinity, tolerations, priorityClassName
```

## Requests vs limits

```bash
# requests = scheduler reservation | limits = kubelet-enforced ceiling
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "200m", memory: "256Mi" }
kubectl describe node | grep -A5 "Allocated resources"  # shows requests consumed
kubectl top pod <pod>                                   # live usage vs limits
```

## Services & networking

```bash
kubectl get endpoints <svc>              # the real pod IPs behind the VIP
kubectl get svc <svc> -o yaml | grep clusterIP
iptables -t nat -L KUBE-SERVICES -n      # kube-proxy's programmed rules
kubectl run -it --rm dns --image=busybox -- nslookup <svc>  # CoreDNS check
```

## Auth quick map

| Identity | Credential | Checked by |
|---|---|---|
| User | client cert / token / OIDC | API server |
| Kubelet, scheduler, controllers | TLS client cert (cluster CA) | API server |
| Pod | ServiceAccount token (auto-mounted) | API server |
| What they can do | RBAC: Role/ClusterRole + Binding | API server |

## Debugging the control plane

```bash
kubectl -v=6 get pods            # see the exact API call failing
systemctl status kubelet         # kubelet alive on the node?
journalctl -u kubelet --since "30 min ago" | tail -50
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health
ls /etc/kubernetes/manifests/    # static pod definitions (self-hosted clusters)
```

## Interview one-liners

- "Docker runs a container; Kubernetes is the OS for a fleet."
- "Desired state in etcd; control loops drive reality toward it."
- "Lose etcd quorum = lose the cluster."
- "CPU limits throttle; memory limits OOM-kill."
- "kubectl only talks to the API server — connection errors are control-plane problems."
