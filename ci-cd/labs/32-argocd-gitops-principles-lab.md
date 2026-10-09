# ArgoCD & GitOps Principles — Hands-On Lab

Work through these exercises in order. Each one has a **goal**, the **commands** to run, the **expected output**, and **why it matters** for interviews.

> You need a Kubernetes cluster (kind, k3d, or minikube works fine) and a GitHub repo you can push to. Never point ArgoCD at production clusters for practice.

---

## Exercise 1: Install ArgoCD on your cluster

**Goal:** Get ArgoCD running and reach the UI.

**Commands:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods   # wait for all Running
# Get the admin password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# Port-forward the UI:
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**Expected output:** All `argocd-*` pods Running; you can log into `https://localhost:8080` as `admin`.

**Why it matters:** Shows you can stand up ArgoCD from scratch — the first thing an interviewer assumes you can do if you claim GitOps experience.

---

## Exercise 2: Create your GitOps repo

**Goal:** Build the Git repo that will be your single source of truth.

**Commands:**
```bash
mkdir -p gitops-demo/apps/hello/overlays/prod gitops-demo/apps/hello/base
cat > gitops-demo/apps/hello/base/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata: {name: hello}
spec:
  replicas: 2
  selector: {matchLabels: {app: hello}}
  template:
    metadata: {labels: {app: hello}}
    spec:
      containers:
      - {name: hello, image: nginx:1.27, ports: [{containerPort: 80}]}
EOF
cat > gitops-demo/apps/hello/base/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata: {name: hello}
spec:
  selector: {app: hello}
  ports: [{port: 80}]
EOF
cat > gitops-demo/apps/hello/overlays/prod/kustomization.yaml <<'EOF'
resources:
- ../../base
EOF
cd gitops-demo && git init && git add . && git commit -m "hello app base + prod overlay" && git push -u origin main
```

**Expected output:** A pushed repo with `base/` manifests and a `prod` Kustomize overlay.

**Why it matters:** The repo layout IS the GitOps design — per-environment overlays are how real teams structure this.

---

## Exercise 3: Create your first Application

**Goal:** Register the app in ArgoCD and watch it sync from Git.

**Commands:**
```bash
argocd login localhost:8080 --username admin --insecure
argocd app create hello-prod \
  --repo https://github.com/<you>/gitops-demo.git \
  --path apps/hello/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace hello-prod \
  --sync-policy automated
argocd app get hello-prod
kubectl -n hello-prod get pods
```

**Expected output:** App shows `Synced` and `Healthy`; two nginx pods run in `hello-prod`.

**Why it matters:** This is the core loop — Git commit → ArgoCD sync → running pods. Everything else builds on it.

---

## Exercise 4: See drift and self-heal in action

**Goal:** Make a manual change in the cluster and watch ArgoCD react.

**Commands:**
```bash
# 1. With automated sync but no self-heal, edit the cluster directly:
kubectl -n hello-prod scale deploy/hello --replicas=5
argocd app get hello-prod   # shows OutOfSync
# 2. Now enable self-heal on the app:
argocd app set hello-prod --self-heal
kubectl -n hello-prod scale deploy/hello --replicas=5
sleep 60; kubectl -n hello-prod get deploy hello -o jsonpath='{.spec.replicas}'
```

**Expected output:** First manual scale flags OutOfSync; after self-heal, the replica count snaps back to 2 within a minute.

**Why it matters:** Drift detection is the GitOps superpower — and the most common thing interviewers ask you to demo.

---

## Exercise 5: Deploy a change through Git (the GitOps way)

**Goal:** Roll out a new version using only a Git commit — no kubectl.

**Commands:**
```bash
cd gitops-demo
sed -i 's/nginx:1.27/nginx:1.28/' apps/hello/base/deployment.yaml
git commit -am "bump nginx to 1.28" && git push
argocd app get hello-prod   # Synced, Health: Progressing -> Healthy
kubectl -n hello-prod rollout status deploy/hello
```

**Expected output:** ArgoCD picks up the commit within ~3 minutes and rolls the Deployment to nginx:1.28 automatically.

**Why it matters:** This is the whole pitch — every prod change is a reviewed commit, and rollbacks are `git revert`.

---

## Exercise 6: Add a sync wave for ordering

**Goal:** Make sure a ConfigMap exists before the Deployment that mounts it.

**Commands:**
```bash
cat > gitops-demo/apps/hello/base/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
data: {GREETING: "hello from gitops"}
EOF
cd gitops-demo && git commit -am "add configmap with sync wave -1" && git push
argocd app sync hello-prod
argocd app get hello-prod
```

**Expected output:** Sync succeeds; the ConfigMap (wave -1) applies before the Deployment.

**Why it matters:** Ordering problems are the #1 ArgoCD beginner pain — sync waves are the standard fix.

---

## Exercise 7: Silence false drift with ignoreDifferences

**Goal:** Add an HPA and stop ArgoCD complaining about replica drift.

**Commands:**
```bash
kubectl -n hello-prod autoscale deploy/hello --cpu-percent=50 --min=2 --max=5
sleep 90; argocd app get hello-prod   # likely OutOfSync on replicas
# Fix: patch the Application to ignore HPA-owned field
argocd app patch hello-prod --patch '{"spec":{"ignoreDifferences":[{"group":"apps","kind":"Deployment","jsonPointers":["/spec/replicas"]}]}}' --type merge
argocd app get hello-prod   # back to Synced
```

**Expected output:** App goes OutOfSync from HPA scaling, then returns to Synced after ignoreDifferences.

**Why it matters:** Every production ArgoCD app needs this — controllers that mutate resources (HPA, cert-manager) create false drift.

---

## Exercise 8: Try an ApplicationSet

**Goal:** Generate per-environment Applications from one template.

**Commands:**
```bash
cat > gitops-demo/applicationset.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: {name: hello-all, namespace: argocd}
spec:
  generators:
  - list:
      elements:
      - {env: dev, ns: hello-dev}
      - {env: prod, ns: hello-prod}
  template:
    metadata: {name: 'hello-{{env}}'}
    spec:
      project: default
      source:
        repoURL: https://github.com/<you>/gitops-demo.git
        path: 'apps/hello/overlays/{{env}}'
      destination: {server: https://kubernetes.default.svc, namespace: '{{ns}}'}
      syncPolicy: {automated: {}}
EOF
kubectl -n argocd apply -f gitops-demo/applicationset.yaml
argocd app list | grep hello-
```

**Expected output:** Two Applications (`hello-dev`, `hello-prod`) appear, generated from the template. (Create the `dev` overlay first or the dev one stays OutOfSync — that's expected.)

**Why it matters:** ApplicationSets are how teams scale from 5 apps to 500 without hand-writing manifests.

---

## Exercise 9: Inspect the diff like an interviewer would

**Goal:** Practice reading ArgoCD's comparison output.

**Commands:**
```bash
# Make a change, then inspect BEFORE syncing:
cd gitops-demo && sed -i 's/replicas: 2/replicas: 4/' apps/hello/base/deployment.yaml && git commit -am "scale to 4" && git push
argocd app diff hello-prod
argocd app get hello-prod   # OutOfSync, diff shows replicas 2 -> 4
argocd app sync hello-prod
```

**Expected output:** `argocd app diff` prints the exact field-level diff; sync applies it.

**Why it matters:** "Walk me through an OutOfSync app" is a classic interview prompt — the diff output is your answer's evidence.

---

## Exercise 10: Roll back with git revert

**Goal:** Prove that rollback in GitOps is just Git.

**Commands:**
```bash
cd gitops-demo && git log --oneline -3
git revert HEAD --no-edit && git push
argocd app get hello-prod
kubectl -n hello-prod get deploy hello -o jsonpath='{.spec.replicas}'
```

**Expected output:** The revert commit syncs automatically; the Deployment returns to its previous state.

**Why it matters:** This one-liner is the strongest GitOps selling point you can demo — audit trail and rollback for free.
