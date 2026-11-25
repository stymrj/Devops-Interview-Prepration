# Kubernetes Autoscaling Deep Dive — Hands-On Lab

Nine drills that turn autoscaling answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Install metrics-server and watch the HPA read live CPU

**Goal:** A working metrics-server and an HPA that shows real CPU utilisation — and proof the HPA is blind without it.

**Steps:**

```bash
# Install metrics-server (kind/minikube: use the insecure-tls variant for local clusters):
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# Confirm the metrics API answers:
kubectl top nodes
kubectl top pods
# Deploy a sample app with CPU requests set (HPA needs requests to compute utilisation):
kubectl create deployment web --image=nginx --replicas=2
kubectl set resources deployment/web --requests=cpu=100m,memory=128Mi
```

**Expected output:** `kubectl top nodes` and `kubectl top pods` return real CPU and memory numbers instead of an error — and one line noting that without metrics-server the HPA would show `<unknown>` forever.

**Why it matters:** This is Q2 made concrete — the HPA is only as good as its metrics pipeline, and the first debug step is always `kubectl top`.

---

## Exercise 2: Create an HPA and drive it with load — watch replicas move

**Goal:** An HPA that scales a deployment from 2 to 6 replicas under load, then back down — with the timing visible.

**Steps:**

```bash
# Create the HPA: scale on 50% average CPU, min 2, max 6:
kubectl autoscale deployment web --cpu-percent=50 --min=2 --max=6
kubectl get hpa -w
# In another terminal, generate load:
kubectl run -i --tty load-generator --image=busybox --rm --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://web; done"
```

**Expected output:** `kubectl get hpa` shows TARGETS climbing past 50%, then REPLICAS stepping up 2 → 4 → 6 — and after you stop the load, replicas stay high for about 5 minutes before stepping down, proving the default scale-down stabilization window.

**Why it matters:** This is Q3 made concrete — the formula is visible in the TARGETS column, and the 5-minute cool-down is something you have to see once to believe.

---

## Exercise 3: Write a behavior block and prove flapping stops

**Goal:** The same deployment under spiky load, first flapping with default behaviour, then steady with a tuned behavior block.

**Steps:**

```bash
# Apply an HPA with slow scale-down and a capped scale-up rate:
cat > hpa-behavior.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
EOF
kubectl apply -f hpa-behavior.yaml
# Hit it with short bursts of load and watch:
kubectl get hpa web -w
```

**Expected output:** Replica count climbs in controlled steps during bursts and does not drop the moment a burst ends — and a note comparing this with the default behaviour, where a single spike added pods that vanished a minute later.

**Why it matters:** This is Q5 made concrete — stabilization windows and scale policies are the answer to flapping, and you can demonstrate both in one terminal.

---

## Exercise 4: Scale on a custom metric — requests per second via Prometheus Adapter

**Goal:** An HPA driven by requests per second instead of CPU, using a Prometheus query as the metric.

**Steps:**

```bash
# Install kube-prometheus-stack and the Prometheus Adapter (helm):
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
helm install prometheus-adapter prometheus-community/prometheus-adapter -n monitoring \
  --set prometheus.url=http://monitoring-kube-prometheus-prometheus.monitoring.svc
# Check the custom metrics API lists your query:
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | head -c 500
```

**Expected output:** The custom metrics API answers and lists metrics — and a written HPA manifest that targets `http_requests_per_second` with `averageValue: "100"`, ready to apply once the app exposes a counter.

**Why it matters:** This is Q4 made concrete — request rate is the metric users actually feel, and the adapter is the bridge that makes the HPA see it.

---

## Exercise 5: Run the VPA in Off mode and read its recommendations

**Goal:** A VPA in recommendation-only mode, producing a sane CPU/memory recommendation for a real workload after a day of traffic.

**Steps:**

```bash
# Install the VPA (upstream manifests):
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
# Create a VPA in Off mode — recommendations only, no evictions:
cat > vpa-off.yaml <<'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
EOF
kubectl apply -f vpa-off.yaml
kubectl describe vpa web
```

**Expected output:** `kubectl describe vpa` shows a `Recommendation` section with target CPU/memory that differs from the current requests — and a one-line judgement on whether the current requests are over- or under-provisioned.

**Why it matters:** This is Q7 and Q9 made concrete — Off mode is how you learn what the VPA thinks before you let it evict anything.

---

## Exercise 6: Install KEDA and scale a worker to zero

**Goal:** A queue worker that scales from zero pods when a message arrives, and back to zero when the queue empties.

**Steps:**

```bash
# Install KEDA:
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda -n keda --create-namespace
# Deploy a worker at 0 replicas with a ScaledObject on a queue (use a local Redis list as the trigger):
cat > scaledobject.yaml <<'EOF'
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker
spec:
  scaleTargetRef:
    name: order-worker
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: redis
      metadata:
        address: redis:6379
        listName: orders
        listLength: "5"
EOF
kubectl apply -f scaledobject.yaml
kubectl get pods -w   # watch: 0 pods while the list is empty
```

**Expected output:** Zero pods while the queue is empty, pods appearing within seconds of pushing messages onto the list, and back to zero after the queue drains — plus a note of the cold-start delay you measured on the first message.

**Why it matters:** This is Q8 made concrete — scale-to-zero is KEDA's killer feature, and the cold-start delay is the reason it stays away from user-facing APIs.

---

## Exercise 7: Simulate the Cluster Autoscaler decision — unschedulable pod, then a new node

**Goal:** A pod that cannot be scheduled, a visible Cluster Autoscaler scale-up event, and a node that disappears again when the pod is deleted.

**Steps:**

```bash
# On a managed cluster with the Cluster Autoscaler enabled, create a pod asking for more CPU than any node has free:
cat > big-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: too-big
spec:
  containers:
    - name: app
      image: nginx
      resources:
        requests:
          cpu: "8"
          memory: 16Gi
EOF
kubectl apply -f big-pod.yaml
kubectl get pods too-big -w        # Pending, then Running once a node appears
kubectl get nodes -w               # a new node joins
kubectl delete pod too-big         # after 10 idle minutes, watch the node leave
```

**Expected output:** The pod stuck in `Pending` with a `FailedScheduling` event, then `Running` once a new node joins — and the Cluster Autoscaler log line showing the scale-up decision and the later scale-down after the node sits idle.

**Why it matters:** This is Q10 made concrete — the autoscaler only acts on unschedulable pods, and the node only leaves when every pod on it can move.

---

## Exercise 8: Break autoscaling on purpose — one unmovable pod blocks scale-down

**Goal:** A node that refuses to scale down because of a single pod, and the fix that unblocks it.

**Steps:**

```bash
# Schedule a pod with a local emptyDir volume that holds data (or a pod without a PodDisruptionBudget on a single-replica Deployment):
kubectl create deployment sticky --image=nginx --replicas=1
# Check what the Cluster Autoscaler says about the node:
kubectl logs -n kube-system deployment/cluster-autoscaler | grep -i "not removable\|unremovable" | head -5
# Fix it: give the workload a PDB and more than one replica so it can be evicted safely:
kubectl scale deployment sticky --replicas=2
cat > pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: sticky-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: sticky
EOF
kubectl apply -f pdb.yaml
```

**Expected output:** The autoscaler log naming the pod as unremovable and the reason — and after the fix, the node becoming eligible for scale-down, with a note on which change unblocked it.

**Why it matters:** This is the Q10 gotcha made concrete — one pinned pod holds a whole node hostage, and the fix is a PDB plus a second replica.

---

## Exercise 9: The sale-day drill — full stack, one runbook, one load test

**Goal:** A written sale-day autoscaling runbook, validated by a load test that proves the HPA, the nodes, and the database all hold at 3x normal traffic.

**Steps:**

```bash
# 1. Baseline: record current HPA maxReplicas, node pool size, DB connection pool size.
kubectl get hpa; kubectl get nodes; kubectl get configmap db-config -o yaml | grep -i pool
# 2. Pre-warm: scale the node pool up an hour before the test (don't let the HPA wait on nodes).
# 3. Run the load test at 3x normal traffic (k6, from the Topic 77 lab) and watch all three layers:
kubectl get hpa -w &
kubectl get nodes -w &
# 4. Write the runbook: HPA settings, pre-warm time, DB pool size, payment-gateway rate limit, alert thresholds.
```

**Expected output:** A one-page runbook with the HPA target, maxReplicas, pre-warm lead time, DB pool size at max replicas, and the "HPA at max replicas for 5 minutes" alert — plus the load-test numbers proving 3x traffic held, or the bottleneck the test exposed.

**Why it matters:** This is Q12 made concrete — autoscaling is a stack, and the drill proves the app, the nodes, and the database scale together instead of failing one at a time.
