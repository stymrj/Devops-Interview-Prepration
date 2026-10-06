# Helm Charts — Hands-On Lab

*8 exercises to make Helm muscle memory. Run on a local cluster (kind, minikube, or Docker Desktop).*

---

## Exercise 1: Scaffold a chart and read the pieces

**Goal:** Understand what `helm create` gives you.

```bash
helm create myapp
tree myapp
cat myapp/Chart.yaml
cat myapp/values.yaml
```

**Expected output:** A directory with `Chart.yaml`, `values.yaml`, `templates/` (deployment, service, ingress, serviceaccount, hpa, tests), and `charts/`. Chart.yaml shows `apiVersion: v2`, a name, and a `0.1.0` version.

**Why it matters:** Interviews ask you to walk a chart's structure from memory. Knowing the scaffold cold is the fastest way.

---

## Exercise 2: Render templates without deploying

**Goal:** See how values flow into templates.

```bash
helm template myrelease ./myapp --set replicaCount=3
helm template myrelease ./myapp --set replicaCount=3 | grep -A2 "replicas:"
```

**Expected output:** Fully rendered Kubernetes YAML printed to stdout — a Deployment with `replicas: 3`, a Service, etc. Nothing is sent to the cluster.

**Why it matters:** `helm template` is your debugging superpower — you catch templating mistakes before they hit the API server.

---

## Exercise 3: Install a release and inspect it

**Goal:** Deploy something real and see how Helm tracks it.

```bash
helm install myapp ./myapp
helm list
helm status myapp
kubectl get secret -l owner=helm
```

**Expected output:** `helm list` shows `myapp` with revision 1, status deployed. `helm status` prints the release notes and rendered resources. A Secret named like `sh.helm.release.v1.myapp.v1` exists — that's where Helm stores the release state.

**Why it matters:** Releases are stored as Secrets in the namespace — there's no Tiller, no server component. That's a classic interview question.

---

## Exercise 4: Upgrade, check history, and roll back

**Goal:** Practice the release lifecycle.

```bash
helm upgrade myapp ./myapp --set image.tag=v2 --wait --timeout 2m
helm history myapp
helm rollback myapp 1
helm history myapp
```

**Expected output:** `helm history` lists revisions 1, 2, then 3 (the rollback). Status of revision 2 shows `superseded`, revision 3 shows `deployed`.

**Why it matters:** Rollback is just re-applying a stored revision. Know the commands cold — upgrades fail in production and this is the recovery path.

---

## Exercise 5: Feel the values precedence yourself

**Goal:** Prove to yourself which override wins.

```bash
cat > prod-values.yaml <<'EOF'
replicaCount: 5
EOF
helm template myapp ./myapp -f prod-values.yaml --set replicaCount=9 | grep -A1 "replicas:"
```

**Expected output:** `replicas: 9` — the `--set` flag beats the `-f` file, which beats the chart's `values.yaml` default.

**Why it matters:** "Which wins?" is a direct interview question. Now you've seen it, not just memorized it.

---

## Exercise 6: Use --atomic and --wait on a failing chart

**Goal:** See how Helm protects you from stuck releases.

```bash
# Break the chart: set an image that doesn't exist
helm upgrade myapp ./myapp --set image.repository=nonexistent/broken \
  --wait --timeout 60s --atomic
helm status myapp
helm list
```

**Expected output:** The upgrade times out waiting for pods that never become ready, and `--atomic` automatically rolls back to the previous revision. No `PENDING_UPGRADE` stuck state — `helm list` still shows the release as deployed.

**Why it matters:** `--atomic` is the one flag that separates people who've run Helm in production from people who've only read about it.

---

## Exercise 7: Debug a templating bug with lint and dry-run

**Goal:** Build the habit of pre-deploy validation.

```bash
helm lint ./myapp
helm install dryrun ./myapp --dry-run --debug | head -40
```

**Expected output:** `helm lint` reports `[INFO]` checks and any warnings. The dry-run shows the full rendered manifest with the debug header, exactly what would be sent to the API server.

**Why it matters:** Lint catches Chart.yaml mistakes; `--dry-run --debug` catches rendering mistakes. Running both before a real deploy is the professional habit.

---

## Exercise 8: Clean up and understand what uninstall removes

**Goal:** See the full lifecycle end.

```bash
helm uninstall myapp
helm list
kubectl get all -l app.kubernetes.io/instance=myapp
```

**Expected output:** `helm list` is empty; all the release's resources (deployment, service) are gone. The release Secret is removed too — `uninstall` wipes Helm's record of the release.

**Why it matters:** `uninstall` deletes everything the release owned, including the stored release history. Know the difference from `rollback`, which preserves history.

---

## Bonus: chart hooks in action

**Goal:** Watch a hook fire exactly once.

```bash
mkdir -p myapp/templates
cat > myapp/templates/pre-install-job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: busybox
          command: ["sh", "-c", "echo running migrations"]
EOF
helm template test ./myapp | grep -B2 "hook"
```

**Expected output:** The Job renders with the `helm.sh/hook: pre-install` annotation. When installed for real, it runs once before other resources and gets deleted on success — it is NOT tracked as part of the release.

**Why it matters:** Hooks are one-shot lifecycle actions, not managed resources. Seeing the annotation is how the concept sticks.
