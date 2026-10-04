# Pods, ReplicaSets & Workload Controllers — Hands-On Lab

*10 exercises to make Pods, controllers, and probes real. Run on any local cluster (kind, minikube, k3d) or a disposable cloud cluster.*

---

## Exercise 1: Inspect a Pod's Shared Network

**Goal:** Prove containers in one Pod share an IP and localhost.

**Commands:**
```bash
kubectl run two-box --image=nginx --image=busybox 2>/dev/null || true
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: two-box }
spec:
  containers:
  - name: web
    image: nginx
  - name: tools
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl get pod two-box -o jsonpath='{.status.podIP}{"\n"}'
kubectl exec two-box -c tools -- wget -qO- http://localhost
```

**Expected output:** One Pod IP, and the busybox container fetches the nginx welcome page over `localhost`.

**Why it matters:** This is the physical proof of the Pod concept — shared network namespace, one IP, port conflicts possible.

---

## Exercise 2: Watch a ReplicaSet Self-Heal

**Goal:** See the reconciliation loop replace a deleted Pod.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata: { name: web-rs }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
      - name: web
        image: nginx
EOF
kubectl get pods -l app=web -o name
kubectl delete pod -l app=web --grace-period=0 --force 2>/dev/null | head -1
kubectl get pods -l app=web
```

**Expected output:** Three Pods, then after the delete the count drops briefly and a new Pod with a different name appears.

**Why it matters:** The ReplicaSet doesn't resurrect the dead Pod — it creates a brand-new one. Pods are disposable; the controller is the durability.

---

## Exercise 3: See Label Adoption in Action

**Goal:** Prove ReplicaSets adopt any Pod matching their selector.

**Commands:**
```bash
kubectl run stray --image=nginx -l app=web
kubectl get rs web-rs -o jsonpath='{.status.replicas}{"\n"}'
kubectl get pods -l app=web --show-labels
```

**Expected output:** Replica count shows 4 (or the ReplicaSet deletes one of the extra Pods to get back to 3) — the stray Pod gets counted, then culled.

**Why it matters:** Ownership is by label matching, not creation. Clean up with `kubectl delete rs web-rs` afterwards so stray Pods don't linger.

---

## Exercise 4: Add an Init Container That Gates Startup

**Goal:** Make a Pod wait for a dependency before starting.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: gated-app }
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'echo "checking db..."; sleep 5; echo "db up"']
  containers:
  - name: app
    image: nginx
EOF
kubectl get pod gated-app -w
```

**Expected output:** Pod shows `Init:0/1` for ~5 seconds, then flips to Running.

**Why it matters:** Init containers gate the app on prerequisites — migrations, DNS, dependency checks — before the first app container ever starts.

---

## Exercise 5: Break a Readiness Probe and Watch Endpoints Shrink

**Goal:** See a Pod removed from Service traffic without being restarted.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: probe-demo, labels: { app: probe } }
spec:
  containers:
  - name: web
    image: nginx
    readinessProbe:
      httpGet: { path: /missing, port: 80 }
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata: { name: probe-svc }
spec:
  selector: { app: probe }
  ports: [{ port: 80 }]
EOF
kubectl get endpoints probe-svc
kubectl describe pod probe-demo | grep -A2 Readiness
```

**Expected output:** `ENDPOINTS` is empty and readiness shows failing — but the Pod stays Running (not restarted). Change the path to `/` and re-apply to watch the endpoint appear.

**Why it matters:** Readiness removes from traffic without killing the container — exactly how rolling updates avoid dropping requests.

---

## Exercise 6: Cause a CrashLoopBackOff on Purpose

**Goal:** Learn the fastest debug path for the most common Pod failure.

**Commands:**
```bash
kubectl run crasher --image=busybox --command -- sh -c "echo boom; exit 1"
sleep 20
kubectl get pod crasher
kubectl logs crasher --previous 2>/dev/null || kubectl logs crasher
kubectl describe pod crasher | grep -A3 "Last State"
```

**Expected output:** `CrashLoopBackOff`, the log line `boom`, and the last-state exit code 1.

**Why it matters:** In interviews you say "check logs and exit code first" — this is the muscle memory for it.

---

## Exercise 7: Run a DaemonSet and Count Node Coverage

**Goal:** Confirm DaemonSet lands exactly one Pod per node.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata: { name: node-agent }
spec:
  selector: { matchLabels: { app: agent } }
  template:
    metadata: { labels: { app: agent } }
    spec:
      containers:
      - name: agent
        image: busybox
        command: ["sleep", "3600"]
EOF
kubectl get nodes --no-headers | wc -l
kubectl get pods -l app=agent --no-headers | wc -l
```

**Expected output:** The two counts match — one agent Pod per node.

**Why it matters:** This is how log shippers and monitoring agents actually deploy — one per machine, automatic as nodes scale.

---

## Exercise 8: Deploy a StatefulSet and Check Stable Identity

**Goal:** See ordered creation and predictable hostnames.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata: { name: db }
spec:
  clusterIP: None
  selector: { app: db }
  ports: [{ port: 5432 }]
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: db }
spec:
  serviceName: db
  replicas: 2
  selector: { matchLabels: { app: db } }
  template:
    metadata: { labels: { app: db } }
    spec:
      containers:
      - name: db
        image: busybox
        command: ["sleep", "3600"]
EOF
kubectl get pods -l app=db
kubectl exec db-0 -- hostname
kubectl delete pod db-0 db-1
kubectl get pods -l app=db
```

**Expected output:** Pods named `db-0`, `db-1` created in order; after deletion they come back with the same names.

**Why it matters:** Stable identity is the whole point of StatefulSets — the names (and volumes, with a claim template) survive restarts.

---

## Exercise 9: Run a Job to Completion and Watch Cleanup

**Goal:** Run work to completion with retry and TTL behavior.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata: { name: hello-job }
spec:
  completions: 3
  parallelism: 1
  backoffLimit: 2
  ttlSecondsAfterFinished: 120
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: hello
        image: busybox
        command: ["sh", "-c", "echo done-$RANDOM"]
EOF
kubectl wait --for=condition=complete job/hello-job --timeout=60s
kubectl logs -l job-name=hello-job
```

**Expected output:** Three successful Pods, three "done-" lines, and the Job auto-deletes ~2 minutes after finishing.

**Why it matters:** Jobs with `ttlSecondsAfterFinished` keep finished work from piling up — a detail interviewers expect on batch workloads.

---

## Exercise 10: Create a CronJob and Verify Its Schedule

**Goal:** Confirm cron scheduling works and Pods spawn on time.

**Commands:**
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata: { name: tick }
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      ttlSecondsAfterFinished: 60
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: tick
            image: busybox
            command: ["date"]
EOF
sleep 75
kubectl get jobs -l job-name 2>/dev/null; kubectl get jobs | grep tick
```

**Expected output:** A new Job named `tick-<timestamp>` appears every minute.

**Why it matters:** CronJobs are the Kubernetes-native answer to "run this nightly" — migrations, backups, reports — and `suspend: true` is how you pause them.

---

## Cleanup

```bash
kubectl delete pod two-box gated-app probe-demo crasher stray 2>/dev/null
kubectl delete rs web-rs 2>/dev/null
kubectl delete ds node-agent 2>/dev/null
kubectl delete sts db 2>/dev/null
kubectl delete svc probe-svc db 2>/dev/null
kubectl delete cronjob tick 2>/dev/null
```
