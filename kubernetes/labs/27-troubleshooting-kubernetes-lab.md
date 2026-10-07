# Hands-on Lab: Troubleshooting Kubernetes

Practice debugging a broken cluster like you'd do in an incident — every exercise breaks something on purpose, then you find and fix it.

**Prerequisites:** a local cluster (kind, minikube, or k3d) and `kubectl` pointed at it.

---

## Exercise 1: Read a CrashLoopBackOff

**Goal:** Learn the logs-first, describe-second workflow.

**Commands:**
```bash
kubectl run crasher --image=busybox --restart=Never -- /bin/sh -c "echo oops && exit 1"
sleep 20
kubectl logs crasher --previous
kubectl describe pod crasher | grep -A 8 "Events:"
```

**Expected output:** Logs show `oops` from the last crash. Events show repeated `BackOff restarting failed container` entries with increasing delay.

**Why it matters:** In real incidents, the previous container's logs hold the error. If you only look at the current (restarting) container, you see nothing.

---

## Exercise 2: Fix an ImagePullBackOff

**Goal:** Diagnose image pull failures from the event messages.

**Commands:**
```bash
kubectl run badimage --image=nginx:does-not-exist-999
kubectl describe pod badimage | grep -B 1 -A 3 "Failed to pull image"
```

**Expected output:** Event shows `Failed to pull image "nginx:does-not-exist-999"` with a manifest-unknown style error from the registry.

**Why it matters:** ImagePullBackOff is a fast win in interviews and incidents — the fix is usually a tag typo or a missing pull secret, and the describe output tells you which.

---

## Exercise 3: Create a Pending pod and find the scheduler's reason

**Goal:** See how the scheduler explains itself.

**Commands:**
```bash
kubectl run hungry --image=nginx --requests='cpu=100000,memory=100Gi'
kubectl describe pod hungry | grep -A 4 "Events:"
```

**Expected output:** A `FailedScheduling` warning: `0/N nodes are available: N Insufficient memory` (or cpu).

**Why it matters:** Pending pods always come with a scheduler verdict in Events. Reading it is the whole skill — the fix (lower requests, fix affinity, scale nodes) follows the verdict.

---

## Exercise 4: Empty endpoints — the silent service bug

**Goal:** Discover how a wrong selector label silently breaks a service.

**Commands:**
```bash
kubectl create deployment websvc --image=nginx
kubectl expose deployment websvc --port=80
kubectl get endpoints websvc
kubectl get pods -l app=websvc --show-labels
```

**Expected output:** `kubectl get endpoints websvc` shows `<none>` — the deployment's labels don't match the service selector you expect. Then fix: `kubectl expose` created selector `app=websvc`; check with `kubectl get svc websvc -o yaml | grep -A 2 selector`.

**Why it matters:** A service with empty endpoints returns connection refused with zero errors on the pod side. Checking endpoints is the fastest way to separate app bugs from selector bugs.

---

## Exercise 5: Exec-and-curl from inside the pod

**Goal:** Practice the local-curl test that splits app vs. network problems.

**Commands:**
```bash
kubectl exec -it deploy/websvc -- curl -s -o /dev/null -w "%{http_code}\n" localhost:80
```

**Expected output:** `200` — nginx answers locally, so if external traffic fails, the problem is the service/ingress, not the app.

**Why it matters:** This one command decides your whole debug direction in under five seconds.

---

## Exercise 6: Debug DNS from a pod

**Goal:** Test cluster DNS the way CoreDNS sees it.

**Commands:**
```bash
kubectl run dns-test --image=busybox --restart=Never -it --rm -- nslookup kubernetes.default
kubectl run dns-test2 --image=busybox --restart=Never -it --rm -- nslookup websvc.default
```

**Expected output:** Both resolve to cluster IPs (`10.x.x.x`). Then try a wrong name (`nslookup nope.default`) and note the `NXDOMAIN`.

**Why it matters:** DNS failures feel like network failures. nslookup from a pod tells you in one step whether it's DNS or the network path.

---

## Exercise 7: Force and diagnose an OOMKill

**Goal:** Watch the kernel kill a container and read the evidence.

**Commands:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: hog
    image: busybox
    command: ["sh", "-c", "yes > /dev/null"]
    resources:
      limits:
        memory: "64Mi"
EOF
sleep 15
kubectl describe pod oom-demo | grep -A 3 "Last State"
```

**Expected output:** `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.

**Why it matters:** Exit 137 is one of the most common interview topics. Seeing it once in a lab makes the answer stick forever.

---

## Exercise 8: Stuck rollout and rollback

**Goal:** Practice the roll-forward vs. roll-back decision under pressure.

**Commands:**
```bash
kubectl create deployment stuck --image=nginx:1.25
kubectl set image deployment/stuck nginx=nginx:does-not-exist-999
kubectl rollout status deployment/stuck --timeout=30s || true
kubectl rollout undo deployment/stuck
kubectl rollout status deployment/stuck
```

**Expected output:** The rollout hangs waiting for the bad image, then `rollout undo` restores the working `1.25` revision and status shows success.

**Why it matters:** Rollback is the incident move. Practicing it when calm means you'll do it without hesitation when it counts.

---

## Exercise 9: Debug with an ephemeral container

**Goal:** Attach a debugging toolbox to a pod you can't exec into.

**Commands:**
```bash
kubectl debug -it deploy/websvc --image=nicolaka/netshoot --target=nginx -- sh
# inside: dig kubernetes.default +short ; curl -s localhost:80 -o /dev/null -w "%{http_code}\n"
```

**Expected output:** You get a shell with `dig`, `curl`, `tcpdump` available, sharing the target pod's network namespace.

**Why it matters:** Production images are often distroless — no shell, no curl. Ephemeral debug containers are how you inspect them without rebuilding.

---

## Exercise 10: Describe a node under pressure

**Goal:** Read node conditions the way you'd triage a struggling worker.

**Commands:**
```bash
kubectl describe node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') | grep -B 1 -A 12 "Conditions:"
kubectl top node
```

**Expected output:** Conditions list `MemoryPressure`, `DiskPressure`, `PIDPressure` — all `False` on a healthy node — plus `Ready: True`. `kubectl top node` shows current usage (needs metrics-server).

**Why it matters:** Node pressure is where pod evictions come from. Knowing where to read it turns "everything is slow" into a concrete resource diagnosis.
