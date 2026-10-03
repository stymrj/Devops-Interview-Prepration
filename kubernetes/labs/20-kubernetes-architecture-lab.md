# Kubernetes Architecture — Hands-On Lab

*10 exercises to make the control plane and node components real. Run on any local cluster (kind, minikube, k3d) or a disposable cloud cluster.*

---

## Exercise 1: Map the Control Plane

**Goal:** See every control plane component actually running as pods.

**Commands:**
```bash
kubectl get pods -n kube-system -o wide
kubectl get nodes
```

**Expected output:** Static pods like `etcd-<node>`, `kube-apiserver-<node>`, `kube-controller-manager-<node>`, `kube-scheduler-<node>`, plus `coredns` and `kube-proxy` on workers.

**Why it matters:** Interviews ask "what runs on the control plane" — now you've seen the actual manifests instead of reciting a diagram.

---

## Exercise 2: Read a Static Pod Manifest

**Goal:** Understand why control plane pods are "static."

**Commands:**
```bash
sudo ls /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | head -40
```

**Expected output:** YAML manifests the kubelet watches directly — no API server involved.

**Why it matters:** This is the bootstrapping trick: the kubelet starts the API server from local files, then the API server starts everything else.

---

## Exercise 3: Check etcd Health

**Goal:** Verify the quorum your whole cluster depends on.

**Commands:**
```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster
```

**Expected output:** `https://127.0.0.1:2379 is healthy: successfully committed proposal`.

**Why it matters:** "Check etcd quorum" is the second step of every control-plane outage runbook — now the command is muscle memory.

---

## Exercise 4: Watch a Deployment Create Its Objects

**Goal:** See the controller chain: Deployment → ReplicaSet → Pods.

**Commands:**
```bash
kubectl create deployment hello --image=nginx:1.25 --replicas=3
kubectl get deploy,rs,pods -l app=hello
```

**Expected output:** 1 Deployment, 1 ReplicaSet, 3 Pods, all linked by owner references.

**Why it matters:** Interviewers ask "what happens on kubectl apply" — you can narrate it because you watched each controller fire.

---

## Exercise 5: Trace Who Scheduled Your Pod

**Goal:** See the scheduler's binding decision in the pod's events.

**Commands:**
```bash
kubectl describe pod -l app=hello | grep -A3 Events
kubectl get pod -l app=hello -o wide   # note the NODE column
```

**Expected output:** `Successfully assigned default/hello-xyz to <node>` event from `default-scheduler`.

**Why it matters:** Separates scheduler (assigns) from kubelet (executes) — the exact trap from the guide.

---

## Exercise 6: Steer the Scheduler with a Taint

**Goal:** Force the scheduler to avoid a node, then watch it obey.

**Commands:**
```bash
kubectl taint nodes <node-name> key=value:NoSchedule
kubectl create deployment avoid --image=nginx:1.25 --replicas=2
kubectl get pods -l app=avoid -o wide
kubectl taint nodes <node-name> key=value:NoSchedule-   # clean up
```

**Expected output:** Pods land on every node except the tainted one.

**Why it matters:** Taints/tolerations are how real clusters reserve nodes for system workloads — and a favorite interview follow-up.

---

## Exercise 7: Inspect kube-proxy's Rules

**Goal:** See how a Service becomes NAT rules on the node.

**Commands:**
```bash
kubectl expose deployment hello --port=80
kubectl get endpoints hello
sudo iptables -t nat -L KUBE-SERVICES -n | grep -i hello
```

**Expected output:** A `KUBE-SVC-xxxx` chain with per-endpoint `KUBE-SEP-xxxx` targets, one per pod IP.

**Why it matters:** Turns "kube-proxy programs routing" from a sentence into something you've actually read line by line.

---

## Exercise 8: Break a Node and Watch the Control Plane React

**Goal:** See the node controller mark a node NotReady and reschedule its pods.

**Commands:**
```bash
# on the node: sudo systemctl stop kubelet
kubectl get nodes -w          # watch status flip to NotReady
kubectl get pods -l app=hello -o wide -w   # watch pods reschedule
# on the node: sudo systemctl start kubelet
```

**Expected output:** Node goes `NotReady` after ~40s of missed heartbeats; pods get evicted and recreated on healthy nodes.

**Why it matters:** This is the self-healing story every interviewer asks for — now it's something you did, not something you read.

---

## Exercise 9: Find the API Server Audit Trail

**Goal:** See that every kubectl action flows through the API server.

**Commands:**
```bash
kubectl delete deployment hello avoid
kubectl get events --sort-by=.lastTimestamp | tail -10
```

**Expected output:** Event stream showing deletions driven by controllers, all recorded by the API server.

**Why it matters:** Reinforces the core architecture rule: everything talks through the API server, and it logs everything.

---

## Exercise 10: Requests vs Limits in the Wild

**Goal:** Watch the scheduler use requests while the kubelet enforces limits.

**Commands:**
```bash
kubectl run hungry --image=nginx:1.25 \
  --requests='cpu=100m,memory=128Mi' --limits='cpu=200m,memory=256Mi'
kubectl describe node | grep -A5 "Allocated resources"
kubectl top pod hungry
```

**Expected output:** The 128Mi request subtracted from node allocatable; `kubectl top` showing live usage against the limit.

**Why it matters:** Requests drive placement, limits cap runtime — the distinction that causes real outages when mixed up.

---

*Clean up with `kubectl delete deployment hello avoid` and `kubectl delete pod hungry service hello` when you're done.*
