# Chaos Engineering in Practice — Hands-On Lab

Nine drills that make chaos answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install Chaos Mesh on a local cluster

**Goal:** Get a chaos tool running and confirm its controller is healthy.

**Commands:**

```bash
kind create cluster --name chaos-lab
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
kubectl get pods -n chaos-mesh
```

**Expected output:** The Chaos Mesh controller, daemon, and dashboard pods all reach Running.

**Why it matters:** You've seen the tooling layer — chaos experiments are just CRDs once this is up.

---

## Exercise 2: Deploy a demo app and define its steady state

**Goal:** Practice the step everyone skips — measuring "healthy" before breaking anything.

**Commands:**

```bash
kubectl create namespace shop
kubectl -n shop create deployment checkout --image=nginx:1.25 --replicas=3
kubectl -n shop expose deployment checkout --port=80
kubectl -n shop run probe --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O /dev/null http://checkout/ && echo OK || echo FAIL; sleep 2; done'
kubectl -n shop logs probe -f | head -20
```

**Expected output:** A steady stream of `OK` lines — that's your baseline steady state.

**Why it matters:** This is the "before" picture every hypothesis is compared against.

---

## Exercise 3: Kill a pod and watch the system self-heal

**Goal:** Run your first real experiment — the simplest one there is.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata: { name: kill-one-checkout, namespace: shop }
spec:
  action: pod-kill
  mode: one
  selector: { namespaces: [shop], labelSelectors: { app: checkout } }
EOF
kubectl -n shop get pods -w
kubectl -n shop logs probe | tail -10
```

**Expected output:** One pod dies and is replaced, and the probe log keeps showing `OK` — the hypothesis holds.

**Why it matters:** Pod-kill is the hello-world of chaos, and the probe log is how you prove recovery, not just assume it.

---

## Exercise 4: Inject network latency and watch for retry storms

**Goal:** See how a slow dependency hurts more than a dead one.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata: { name: slow-payment, namespace: shop }
spec:
  action: delay
  mode: all
  selector: { namespaces: [shop], labelSelectors: { app: checkout } }
  delay: { latency: "500ms" }
  duration: "2m"
EOF
kubectl -n shop logs probe | tail -20
kubectl delete networkchaos slow-payment -n shop
```

**Expected output:** Responses slow down during the two minutes, then recover when the experiment ends — and the probe shows whether retries made it worse.

**Why it matters:** This is the latency experiment from Q8, and it teaches you to look for retry amplification.

---

## Exercise 5: Stress CPU on one pod and watch scheduling react

**Goal:** Test how the system behaves when a pod gets greedy.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata: { name: cpu-hog, namespace: shop }
spec:
  mode: one
  selector: { namespaces: [shop], labelSelectors: { app: checkout } }
  stressors:
    cpu: { workers: 2, load: 80 }
  duration: "2m"
EOF
kubectl -n shop top pods
kubectl delete stresschaos cpu-hog -n shop
```

**Expected output:** One pod's CPU spikes while the others stay normal, and the probe keeps succeeding — noisy-neighbour contained.

**Why it matters:** CPU stress is how you prove resource limits actually protect neighbouring workloads.

---

## Exercise 6: Simulate an AZ failure by cordoning a node

**Goal:** Rehearse the multi-AZ hypothesis without a real multi-AZ bill.

**Commands:**

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl cordon "$NODE"
kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data
kubectl -n shop get pods -o wide
kubectl -n shop logs probe | tail -5
kubectl uncordon "$NODE"
```

**Expected output:** Pods reschedule onto the remaining nodes and the probe keeps returning `OK`.

**Why it matters:** This is the one-zone experiment from Q5, run by hand — and the drain is exactly what a real AZ loss looks like to the scheduler.

---

## Exercise 7: Run a Litmus experiment from the hub

**Goal:** See the second major tool and its pre-built experiment library.

**Commands:**

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm install litmus litmuschaos/litmus -n litmus --create-namespace
kubectl get pods -n litmus
# From the Litmus portal: pick pod-delete from the ChaosHub, target the shop namespace,
# set the blast radius to one pod, and run it against the checkout deployment.
kubectl -n shop get pods -w
```

**Expected output:** The Litmus portal shows the experiment run, and one checkout pod is deleted and recreated.

**Why it matters:** You've now run two engines — Chaos Mesh's YAML and Litmus's hub — and can compare them from experience.

---

## Exercise 8: Set an abort condition and trigger it on purpose

**Goal:** Practice the safety mechanism that separates chaos from an outage.

**Commands:**

```bash
# Start a pod-kill experiment, then deliberately break the steady state:
kubectl apply -f - <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata: { name: kill-all-checkout, namespace: shop }
spec:
  action: pod-kill
  mode: all
  selector: { namespaces: [shop], labelSelectors: { app: checkout } }
EOF
kubectl -n shop logs probe | tail -5   # FAIL lines appear — abort threshold crossed
kubectl delete podchaos kill-all-checkout -n shop   # the manual kill switch
kubectl -n shop logs probe | tail -5   # OK lines return as pods recover
```

**Expected output:** The probe shows failures during the experiment, and recovery once you delete the experiment — the abort loop, done by hand.

**Why it matters:** This is the auto-abort answer from Q9, practised manually so you can describe it as something you've done, not read.

---

## Exercise 9: Run a mini game day and write the debrief

**Goal:** Close the loop — the debrief is the deliverable, not the experiment.

**Commands:**

```bash
# Announce in your team channel: "Game day 15:00 — checkout resilience, scenario undisclosed."
# Inject a combined failure: kill one pod AND add latency to the rest.
kubectl apply -f - <<'EOF'
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata: { name: gameday-kill, namespace: shop }
spec:
  action: pod-kill
  mode: one
  selector: { namespaces: [shop], labelSelectors: { app: checkout } }
EOF
kubectl -n shop logs probe | tail -10
kubectl delete podchaos gameday-kill -n shop
# Debrief template — fill it in honestly:
cat <<'EOF'
Game day findings:
1. Detected in ___ minutes via ___ (alert/dashboard/user report?)
2. Mitigated by ___ in ___ minutes.
3. Gaps found: ___ (missing runbook? slow dashboard? unclear owner?)
4. Follow-up tickets: ___
EOF
```

**Expected output:** A filled-in debrief with at least one real gap and a follow-up ticket — that document is the game day's output.

**Why it matters:** This is the Q10 answer in action — the debrief's gaps list is what interviewers want to hear about.
