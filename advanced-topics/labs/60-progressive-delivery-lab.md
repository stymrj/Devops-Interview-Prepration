# Progressive Delivery — Hands-On Lab

Nine drills that make progressive delivery answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install Argo Rollouts on a local cluster

**Goal:** Get the rollout controller running.

**Commands:**

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl get pods -n argo-rollouts
```

**Expected output:** The `argo-rollouts` controller pod is Running.

**Why it matters:** You've seen the controller that owns traffic decisions — the difference between a Deployment and a Rollout stops being theory.

---

## Exercise 2: Deploy the demo app as a Rollout

**Goal:** Replace a Deployment with a Rollout resource.

**Commands:**

```bash
kubectl argo rollouts create -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/basic/rollout.yaml -n demo 2>/dev/null || kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: web, namespace: demo }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers: [{ name: web, image: nginx:1.25, ports: [{ containerPort: 80 }] }]
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: { duration: 1m }
      - setWeight: 100
EOF
kubectl argo rollouts get rollout web -n demo
```

**Expected output:** The rollout shows as Healthy with the canary/stable split visible.

**Why it matters:** The Rollout resource is the unit of progressive delivery — this is the object interviewers ask you to describe.

---

## Exercise 3: Run a manual canary step by step

**Goal:** Feel weight shifts and pauses.

**Commands:**

```bash
kubectl argo rollouts set image web web=nginx:1.26 -n demo
kubectl argo rollouts get rollout web -n demo   # watch it pause at 20%
kubectl argo rollouts promote web -n demo       # step it forward by hand
```

**Expected output:** The rollout pauses at the first weight, then advances only after you promote it.

**Why it matters:** "Pause between steps" is the safety mechanism — doing it once makes the canary story credible.

---

## Exercise 4: Write an AnalysisTemplate against Prometheus

**Goal:** Make promotion depend on a real metric.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate, namespace: demo }
spec:
  args: [{ name: service-name }]
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status=~"2.."}[2m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
EOF
kubectl get analysistemplate success-rate -n demo
```

**Expected output:** The template exists and the query parses — the bar for promotion is now code, not a gut feeling.

**Why it matters:** AnalysisTemplates are the answer to "how do you decide to promote?" — and they live in Git, reviewed like code.

---

## Exercise 5: Force a failed analysis and watch the auto-rollback

**Goal:** See the safety net actually catch something.

**Commands:**

```bash
# point the template at a metric that will fail, or set successCondition absurdly high, then trigger a rollout
kubectl argo rollouts set image web web=nginx:1.27 -n demo
kubectl argo rollouts get rollout web -n demo --watch   # watch it abort and revert
kubectl get svc web-stable -n demo                      # stable still serves 100%
```

**Expected output:** The rollout aborts, the canary scales down, and the stable service serves all traffic — no redeploy needed.

**Why it matters:** "Rollback is traffic, not a redeploy" is the line that lands — once you've watched it happen, you say it with conviction.

---

## Exercise 6: Install Flagger with an Istio or NGINX provider

**Goal:** Get the mesh-side controller running.

**Commands:**

```bash
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/deployment.yaml
kubectl -n flagger-system get pods
```

**Expected output:** Flagger and its load-tester pods are Running in `flagger-system`.

**Why it matters:** Flagger's control point is the mesh, not the workload — installing it makes the Argo-vs-Flagger distinction concrete.

---

## Exercise 7: Run a Flagger canary with a load test webhook

**Goal:** Shift traffic with automated load generation.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: flagger.app/v1beta1
kind: Canary
metadata: { name: web, namespace: demo }
spec:
  targetRef: { apiVersion: apps/v1, kind: Deployment, name: web }
  provider: istio
  service: { port: 80 }
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange: { min: 99 }
      interval: 1m
    webhooks:
    - name: load-test
      type: rollout
      url: http://flagger-loadtester.flagger-system/
      metadata: { cmd: "hey -z 1m -q 10 -c 2 http://web-canary.demo:80/" }
EOF
kubectl get canary web -n demo --watch
```

**Expected output:** The canary weight climbs in 10-point steps while the load test runs, then promotes or aborts on the metrics.

**Why it matters:** Webhooks are how real teams gate promotion on load tests — this is the advanced detail that separates a good answer from a generic one.

---

## Exercise 8: Do a blue-green with Argo Rollouts

**Goal:** Feel the instant-rollback path.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: web-bg, namespace: demo }
spec:
  replicas: 2
  selector: { matchLabels: { app: web-bg } }
  template:
    metadata: { labels: { app: web-bg } }
    spec:
      containers: [{ name: web, image: nginx:1.25, ports: [{ containerPort: 80 }] }]
  strategy:
    blueGreen:
      activeService: web-bg-active
      previewService: web-bg-preview
      autoPromotionEnabled: false
EOF
kubectl argo rollouts promote web-bg -n demo   # flip only when you say so
```

**Expected output:** The preview service serves the new version for inspection, and traffic flips only on promote.

**Why it matters:** Blue-green's selling point is instant rollback by traffic flip — and `autoPromotionEnabled: false` is the manual gate interviewers love to ask about.

---

## Exercise 9: Break it — canary at zero traffic, all checks green

**Goal:** Practice the classic false-green debugging path.

**Commands:**

```bash
# misconfigure the canary service selector so the canary pods get no traffic, trigger a rollout
kubectl argo rollouts get rollout web -n demo
kubectl get endpoints web-canary -n demo   # empty? that's the bug
kubectl get analysisrun -n demo            # passed — with zero samples
# fix the selector, re-run, and confirm the analysis now sees real samples
```

**Expected output:** The analysis passed with no traffic at all; after fixing the selector, the same analysis measures real canary samples.

**Why it matters:** "Green dashboard, failing canary" is the debugging question that ends progressive delivery interviews — a rehearsed false-green drill is a ready-made answer.

---
