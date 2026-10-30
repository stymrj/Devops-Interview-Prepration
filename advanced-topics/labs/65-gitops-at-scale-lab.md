# GitOps at Scale — Hands-On Lab

Nine drills that turn GitOps-at-scale answers into real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install ArgoCD and enable the ApplicationSet controller

**Goal:** Get a working ArgoCD where ApplicationSets actually generate Applications.

**Commands:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods | grep applicationset
# Port-forward the UI
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

**Expected output:** An `argocd-applicationset-controller` pod running, and the UI reachable at https://localhost:8080.

**Why it matters:** This is the controller from Q3 — everything else in this lab depends on it running.

---

## Exercise 2: Generate Applications from a List

**Goal:** Stamp out one Application per environment from a hardcoded list.

**Commands:**

```bash
cat > appset-list.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: demo-list, namespace: argocd }
spec:
  generators:
    - list:
        elements:
          - env: staging
          - env: production
  template:
    metadata: { name: 'demo-{{env}}' }
    spec:
      project: default
      source: { repoURL: 'https://github.com/argoproj/argocd-example-apps', targetRevision: HEAD, path: guestbook }
      destination: { server: 'https://kubernetes.default.svc', namespace: 'demo-{{env}}' }
EOF
kubectl apply -f appset-list.yaml
kubectl -n argocd get applications | grep demo-
```

**Expected output:** Two Applications, `demo-staging` and `demo-production`.

**Why it matters:** This is the List generator from Q4, in its simplest working form.

---

## Exercise 3: Generate Applications from Git directories

**Goal:** Make onboarding a service as simple as adding a folder.

**Commands:**

```bash
mkdir -p services/{payments,orders,inventory}
for s in payments orders inventory; do
  cp -r /tmp/guestbook-chart services/$s/ 2>/dev/null || echo "chart placeholder" > services/$s/Chart.yaml
done
git add services && git commit -m "Add three services" && git push
cat > appset-git.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: services, namespace: argocd }
spec:
  generators:
    - git:
        repoURL: 'https://github.com/<you>/services-repo'
        revision: HEAD
        directories:
          - path: services/*
  template:
    metadata: { name: '{{path.basename}}' }
    spec:
      project: default
      source: { repoURL: 'https://github.com/<you>/services-repo', targetRevision: HEAD, path: '{{path}}' }
      destination: { server: 'https://kubernetes.default.svc', namespace: '{{path.basename}}' }
EOF
kubectl apply -f appset-git.yaml
```

**Expected output:** Three Applications, one per directory, appearing without any per-service YAML.

**Why it matters:** This is the Git directories generator from Q5 — the onboarding story you'll pitch in interviews.

---

## Exercise 4: Generate Applications from Git files with per-service config

**Goal:** Give each service its own parameters from a config file.

**Commands:**

```bash
mkdir -p apps-config
cat > apps-config/payments.json <<'EOF'
{ "service": "payments", "replicas": 3, "chartVersion": "1.4.0" }
EOF
cat > appset-files.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: services-files, namespace: argocd }
spec:
  generators:
    - git:
        repoURL: 'https://github.com/<you>/services-repo'
        revision: HEAD
        files:
          - path: "apps-config/*.json"
  template:
    metadata: { name: '{{service}}' }
    spec:
      project: default
      source:
        repoURL: 'https://github.com/<you>/charts'
        targetRevision: '{{chartVersion}}'
        path: '{{service}}'
        helm:
          parameters:
            - name: replicaCount
              value: '{{replicas}}'
      destination: { server: 'https://kubernetes.default.svc', namespace: '{{service}}' }
EOF
kubectl apply -f appset-files.yaml
```

**Expected output:** A `payments` Application with replicaCount set to 3 from the JSON file.

**Why it matters:** This is the files mode from Q5 — config-driven generation instead of one template for all.

---

## Exercise 5: Cross services with clusters using the Matrix generator

**Goal:** Build the services × clusters fleet from Q7 with one ApplicationSet.

**Commands:**

```bash
cat > appset-matrix.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: fleet, namespace: argocd }
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: 'https://github.com/<you>/services-repo'
              revision: HEAD
              directories:
                - path: services/*
          - clusters:
              selector:
                matchLabels:
                  environment: production
  template:
    metadata: { name: '{{path.basename}}-{{name}}' }
    spec:
      project: default
      source: { repoURL: 'https://github.com/<you>/services-repo', targetRevision: HEAD, path: '{{path}}' }
      destination: { server: '{{server}}', namespace: '{{path.basename}}' }
EOF
kubectl apply -f appset-matrix.yaml
kubectl -n argocd get applications | grep -c "payments-"
```

**Expected output:** One Application per service per production cluster — count equals services × matching clusters.

**Why it matters:** This is the Matrix generator from Q7, the core answer to the 200-services scenario in Q12.

---

## Exercise 6: Per-environment values files from one template

**Goal:** Prove one template can serve staging and prod differently.

**Commands:**

```bash
cat > appset-values.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: env-values, namespace: argocd }
spec:
  generators:
    - list:
        elements:
          - env: staging
          - env: production
  template:
    metadata: { name: 'shop-{{env}}' }
    spec:
      project: default
      source:
        repoURL: 'https://github.com/<you>/charts'
        targetRevision: HEAD
        path: shop
        helm:
          valueFiles:
            - 'values-{{env}}.yaml'
      destination: { server: 'https://kubernetes.default.svc', namespace: 'shop-{{env}}' }
EOF
kubectl apply -f appset-values.yaml
argocd app get shop-staging | grep -i "values-staging"
```

**Expected output:** Two Applications whose only difference is the values file the template selected.

**Why it matters:** This is the no-fork rule from Q8 — differences as parameters, not copied templates.

---

## Exercise 7: Progressive syncs — roll a template change out in batches

**Goal:** Change a template and watch it reach apps in stages, not all at once.

**Commands:**

```bash
cat > appset-progressive.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: { name: fleet-progressive, namespace: argocd }
spec:
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - { key: env, operator: In, values: [staging] }
        - matchExpressions:
            - { key: env, operator: In, values: [production] }
  generators:
    - list:
        elements:
          - env: staging
          - env: production
  template:
    metadata:
      name: 'shop-{{env}}'
      labels: { env: '{{env}}' }
    spec:
      project: default
      source: { repoURL: 'https://github.com/<you>/charts', targetRevision: HEAD, path: shop }
      destination: { server: 'https://kubernetes.default.svc', namespace: 'shop-{{env}}' }
EOF
kubectl apply -f appset-progressive.yaml
kubectl -n argocd get applicationset fleet-progressive -o yaml | grep -A5 "status:"
```

**Expected output:** The ApplicationSet status shows staging synced first, production waiting for the next step.

**Why it matters:** This is the safe-rollout answer from Q9 — batches, not a fleet-wide big bang.

---

## Exercise 8: Keep secrets out of generated Applications

**Goal:** Wire a secret reference that the ApplicationSet never sees the value of.

**Commands:**

```bash
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/main/deploy/crds/bundle.yaml
cat > external-secret.yaml <<'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-creds, namespace: payments }
spec:
  secretStoreRef: { name: vault-backend, kind: ClusterSecretStore }
  target: { name: db-creds }
  data:
    - secretKey: password
      remoteRef: { key: payments/db, property: password }
EOF
kubectl apply -f external-secret.yaml
kubectl -n payments get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

**Expected output:** A real Kubernetes Secret materializes in the cluster while Git holds only the reference.

**Why it matters:** This is the secrets rule from Q10 — parameters are for config, never credentials.

---

## Exercise 9: Debug a generator that matches nothing

**Goal:** Practice the two-half debugging split from Q11.

**Commands:**

```bash
# Break a selector on purpose, then inspect
kubectl -n argocd get applicationset fleet -o yaml | grep -A10 "status:"
argocd appset get fleet
# Fix the label selector, re-apply, and confirm the count recovers
kubectl -n argocd get applications -l argocd.argoproj.io/application-set-name=fleet | wc -l
# Check for hand-edited Applications that drifted from the template
argocd app diff fleet-payments-prod || echo "No drift: generated app matches template"
```

**Expected output:** Zero generated Applications while the selector is wrong, the full count after the fix, and no drift on a template-owned app.

**Why it matters:** This is Q11 in action — wrong count means generator trouble, failing apps mean template trouble.

---
