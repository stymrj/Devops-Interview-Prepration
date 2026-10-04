# Deployments and Rollout Strategies — Hands-On Lab

Practical exercises for topic 22. You need a working cluster (minikube, kind, or any cloud cluster) and `kubectl` configured.

---

## Exercise 1: Create your first Deployment

**Goal:** Deploy a simple app and inspect what the Deployment creates.

**Commands:**

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deploy web
kubectl get rs
kubectl get pods -l app=web
```

**Expected output:** One Deployment, one ReplicaSet owned by it, and three running nginx Pods. `kubectl get rs` shows the ReplicaSet name starts with the Deployment's name plus a hash.

**Why it matters:** Shows the ownership chain Deployment → ReplicaSet → Pods, which explains why you manage updates at the Deployment level.

---

## Exercise 2: Break the selector and fix it

**Goal:** See what happens when a Deployment's selector doesn't match the template labels.

**Commands:**

```bash
kubectl create deployment broken --image=nginx:1.25 --dry-run=client -o yaml > /tmp/broken.yaml
# edit: change template labels to app=wrong, keep selector as app=broken
kubectl apply -f /tmp/broken.yaml
kubectl get pods -l app=wrong
kubectl describe deploy broken
```

**Expected output:** The Deployment creates zero Pods and reports an error like "selector does not match template labels". Fixing the labels makes Pods appear.

**Why it matters:** Selector/template mismatch is the most common beginner Deployment bug — and interviewers love asking about it.

---

## Exercise 3: Trigger a rolling update and watch it happen

**Goal:** Change the image and observe the rolling update in real time.

**Commands:**

```bash
kubectl set image deployment/web nginx=nginx:1.26
kubectl rollout status deployment/web
kubectl get pods -l app=web -w
```

**Expected output:** `rollout status` shows progress ("Waiting for deployment spec update to be observed..."), and the watch shows new Pods appearing with the new hash while old ones terminate one by one.

**Why it matters:** You see the scale-up/scale-down dance the controller performs, which is exactly what Q5 in the guide describes.

---

## Exercise 4: Tune maxSurge and maxUnavailable

**Goal:** Feel the difference between zero-downtime and resource-saving settings.

**Commands:**

```bash
kubectl patch deploy web -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":2}}}}'
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

**Expected output:** The update completes without any Pod ever going unavailable, but at one point you have 5 Pods running for a 3-replica Deployment. Repeat with `maxSurge: 0` and notice the update is slower and old Pods die before new ones exist.

**Why it matters:** These two fields are the knobs interviewers ask you to explain — this exercise makes the trade-off tangible.

---

## Exercise 5: Add a readiness probe and break it on purpose

**Goal:** Prove that readiness probes gate the rollout.

**Commands:**

```bash
kubectl patch deploy web --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"httpGet":{"path":"/nonexistent","port":80},"periodSeconds":5}}]'
kubectl set image deployment/web nginx=nginx:1.28
timeout 60 kubectl rollout status deployment/web
kubectl get pods -l app=web
```

**Expected output:** The rollout stalls — new Pods start but never become ready, so old Pods stay alive and serving traffic. `timeout` kills the status command after 60s; the old version is still untouched.

**Why it matters:** This is the single most important rollout safety mechanism. A bad readiness probe fails closed (stall) instead of failing open (outage).

---

## Exercise 6: Roll back a bad release

**Goal:** Use revision history to recover from a broken deploy.

**Commands:**

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl get pods -l app=web
```

**Expected output:** History lists revisions; undo scales the previous ReplicaSet back up and terminates the bad one; `rollout status` confirms completion and Pods return to the old image.

**Why it matters:** Practicing undo under no pressure is what lets you do it calmly in a real incident — and it's a standard interview follow-up question.

---

## Exercise 7: Pause and resume a rollout

**Goal:** Freeze an update mid-flight to inspect it.

**Commands:**

```bash
kubectl set image deployment/web nginx=nginx:1.29 &
kubectl rollout pause deployment/web
kubectl get rs
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
```

**Expected output:** After pause, both old and new ReplicaSets exist with partial counts and nothing moves. Resume completes the update. `kubectl get rs` during the pause shows the split clearly.

**Why it matters:** Pause is your emergency brake when a rollout looks wrong but you haven't diagnosed it yet.

---

## Exercise 8: Run a canary with two Deployments and one Service

**Goal:** Send a slice of traffic to a new version without a mesh.

**Commands:**

```bash
kubectl create deploy web-stable --image=nginx:1.25 --replicas=9
kubectl create deploy web-canary --image=nginx:1.26 --replicas=1
kubectl label deploy web-stable app=web
kubectl label deploy web-canary app=web
kubectl expose deployment web-stable --name=web-svc --port=80 --target-port=80
kubectl get endpoints web-svc
```

**Expected output:** The Service has 10 endpoints — 9 from stable, 1 from canary. Repeated `curl` requests to the Service hit the canary roughly 1 in 10 times (check via different `Server` headers or pod names in logs).

**Why it matters:** This is the classic no-mesh canary pattern interviewers ask you to whiteboard.

---

## Exercise 9: Debug a stuck rollout

**Goal:** Diagnose a progressDeadlineExceeded scenario end to end.

**Commands:**

```bash
kubectl create deploy stuck --image=nginx:does-not-exist --replicas=2
timeout 90 kubectl rollout status deployment/stuck
kubectl get pods -l app=stuck
kubectl describe pod -l app=stuck | grep -A5 Events
kubectl rollout undo deployment/stuck
```

**Expected output:** Status times out waiting; Pods are in ImagePullBackOff; describe shows "Failed to pull image". Undo rolls back to... nothing (first revision is broken), so you learn undo can't help a broken first deploy — you must fix the image and re-apply.

**Why it matters:** Teaches the triage flow from Q14 and the edge case that undo needs a good revision to exist.

---

## Exercise 10: Inspect the Deployment resource fully

**Goal:** Read a real Deployment YAML and know every field.

**Commands:**

```bash
kubectl get deploy web -o yaml | less
kubectl describe deploy web
```

**Expected output:** You can point to selector, template, strategy, revisionHistoryLimit, and the `deployment.kubernetes.io/revision` annotation on each ReplicaSet. Describe shows Conditions (Available, Progressing) and the rollout Events.

**Why it matters:** Interviews sometimes ask you to read a Deployment spec cold — after this exercise, nothing in it surprises you.
