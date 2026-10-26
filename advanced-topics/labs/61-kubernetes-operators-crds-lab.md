# Kubernetes Operators & CRDs — Hands-On Lab

Nine drills that make operator answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install a CRD and create a custom resource

**Goal:** See the API server serve a brand-new resource type.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: { name: webapps.example.com }
spec:
  group: example.com
  scope: Namespaced
  names: { plural: webapps, singular: webapp, kind: WebApp }
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas: { type: integer, minimum: 1 }
                image: { type: string }
EOF
kubectl apply -f - <<'EOF'
apiVersion: example.com/v1
kind: WebApp
metadata: { name: demo, namespace: default }
spec: { replicas: 2, image: nginx:1.25 }
EOF
kubectl get webapps
```

**Expected output:** The `demo` WebApp appears in `kubectl get webapps`, proving the API server now serves your type.

**Why it matters:** You've felt the core trick of operators — extending the API itself — instead of just reading about it.

---

## Exercise 2: Watch schema validation reject a bad object

**Goal:** Prove the API server validates before your code ever runs.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: example.com/v1
kind: WebApp
metadata: { name: bad, namespace: default }
spec: { replicas: "three", image: nginx:1.25 }
EOF
```

**Expected output:** A validation error rejecting `replicas` as the wrong type.

**Why it matters:** This is the interview answer to "why CRDs over ConfigMaps" — validation at the API server, for free.

---

## Exercise 3: Scaffold an operator with Kubebuilder

**Goal:** Generate a real operator project skeleton.

**Commands:**

```bash
kubebuilder init --domain example.com --repo example.com/webapp-operator
kubebuilder create api --group example.com --version v1 --kind WebApp --resource --controller
make manifests && make install
kubectl get crds | grep example.com
```

**Expected output:** A Go project with `api/`, `controllers/` and `config/` dirs, and the CRD installed in the cluster.

**Why it matters:** You can now describe the standard project layout from memory, which is exactly what interviewers probe.

---

## Exercise 4: Implement a reconcile that owns a Deployment

**Goal:** Make the controller create what the custom resource declares.

**Commands:**

```bash
# In controllers/webapp_controller.go, reconcile to this shape:
# 1. Fetch the WebApp by NamespacedName
# 2. Build a Deployment with spec.replicas and spec.image
# 3. controllerutil.SetControllerReference(webapp, deploy, r.Scheme)
# 4. Create it if missing, update it if it drifted
make run
kubectl apply -f config/samples/example.com_v1_webapp.yaml
kubectl get deployments
```

**Expected output:** A Deployment appears with the replica count and image from the WebApp spec.

**Why it matters:** SetControllerReference is the ownership link that makes garbage collection and watch events work — the heart of the pattern.

---

## Exercise 5: Prove level-based reconciliation survives a restart

**Goal:** Show the controller recovers state instead of needing event history.

**Commands:**

```bash
kubectl scale deployment demo --replicas=5   # drift it by hand
# Kill the controller process (Ctrl-C on make run), then restart it
make run
kubectl get deployments demo -o jsonpath='{.spec.replicas}'
```

**Expected output:** The replica count returns to the WebApp spec value after restart — no manual repair.

**Why it matters:** This is the live demo of "level-based, not edge-based" and why controllers are self-healing.

---

## Exercise 6: Add a status subresource and conditions

**Goal:** Report observed state separately from desired state.

**Commands:**

```bash
# In api/v1/webapp_types.go add:
#   type WebAppStatus struct {
#     Phase string `json:"phase,omitempty"`
#     ReadyReplicas int32 `json:"readyReplicas,omitempty"`
#   }
# In the controller, after reconciling, set status and call r.Status().Update(ctx, webapp)
make manifests && make run
kubectl get webapp demo -o jsonpath='{.status}'
```

**Expected output:** The WebApp shows a `phase` and `readyReplicas` in status, distinct from spec.

**Why it matters:** Separating spec from status is the API contract interviewers check first — and kubectl users see real state, not guesses.

---

## Exercise 7: Add a finalizer and watch deletion pause

**Goal:** See the Terminating gate in action.

**Commands:**

```bash
# In the controller: add the finalizer on first reconcile,
# and on deletion (DeletionTimestamp set) run cleanup, then remove it.
kubectl apply -f config/samples/example.com_v1_webapp.yaml
kubectl delete webapp demo
kubectl get webapp demo   # while cleanup runs
```

**Expected output:** The object lingers in Terminating until cleanup finishes, then disappears.

**Why it matters:** You've seen why finalizers exist — external cleanup must complete before the API object vanishes.

---

## Exercise 8: Break RBAC on purpose and read the error

**Goal:** Learn the most common silent operator failure.

**Commands:**

```bash
# Remove the deployments permission from config/rbac/role.yaml, reinstall, restart:
make manifests && make install && make run
kubectl apply -f config/samples/example.com_v1_webapp.yaml
kubectl logs -l app=webapp-operator 2>/dev/null || echo "check controller logs"
# Restore the permission marker: //+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

**Expected output:** Reconcile errors like `deployments.apps is forbidden` in the controller logs.

**Why it matters:** Missing RBAC is the first thing you check on a stuck operator — this drill makes that reflex automatic.

---

## Exercise 9: Operate a real operator end to end

**Goal:** Run a production-grade operator and drive it through its own API.

**Commands:**

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
kubectl apply -f - <<'EOF'
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: pg-demo, namespace: default }
spec:
  instances: 3
  storage: { size: 1Gi }
EOF
kubectl get cluster pg-demo -w
```

**Expected output:** A 3-instance Postgres cluster comes up, managed entirely through the `Cluster` custom resource.

**Why it matters:** This is the story you tell in interviews — you didn't just build a toy operator, you've driven a real one in production shape.
