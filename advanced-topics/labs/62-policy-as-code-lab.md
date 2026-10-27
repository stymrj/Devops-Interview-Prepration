# Policy as Code (OPA & Kyverno) — Hands-On Lab

Nine drills that make policy answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install Kyverno and see it admit a clean Pod

**Goal:** Get a policy engine running in a cluster and watch it pass good workloads.

**Commands:**

```bash
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl get pods -n kyverno
kubectl run good --image=nginx:1.25 --labels team=platform
kubectl get pod good
```

**Expected output:** The Kyverno pods run, and the clean Pod is created without complaint.

**Why it matters:** You've seen the baseline — a policy engine that stays quiet when nothing is wrong.

---

## Exercise 2: Enforce a require-labels policy and watch a rejection

**Goal:** Feel an admission denial from the developer's side.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-team-label }
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-label
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "Every Pod needs the team label so we know who owns it."
        pattern:
          metadata:
            labels:
              team: "?*"
EOF
kubectl run bad --image=nginx:1.25
```

**Expected output:** The Pod creation is rejected with your policy message about the missing team label.

**Why it matters:** This is the exact denial flow you describe in interviews — and why the message wording matters.

---

## Exercise 3: Block the latest tag with a second rule

**Goal:** Stack rules in one policy and see which one fires.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: block-latest-tag }
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-latest
      match: { any: [{ resources: { kinds: [Pod] } }] }
      validate:
        message: "The latest tag is not allowed — pin a version."
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.containers[].image[] }}"
                operator: AnyIn
                value: ["*:latest"]
EOF
kubectl run latest --image=nginx:latest --labels team=platform
```

**Expected output:** Rejection naming the latest-tag rule, even though the label is fine.

**Why it matters:** You can now explain how multiple rules in one policy are evaluated independently.

---

## Exercise 4: Run a policy in Audit mode and read the violations

**Goal:** Practice the safe rollout: audit before enforce.

**Commands:**

```bash
kubectl patch clusterpolicy require-team-label --type=merge \
  -p '{"spec":{"validationFailureAction":"Audit"}}'
kubectl run audit-test --image=nginx:1.25
kubectl get policyreport -A | head
kubectl describe policyreport -n default | grep -A3 -i violation
kubectl patch clusterpolicy require-team-label --type=merge \
  -p '{"spec":{"validationFailureAction":"Enforce"}}'
```

**Expected output:** The Pod is created, but a PolicyReport records the violation — audit logs without blocking.

**Why it matters:** Audit-first is the rollout answer interviewers want, and now you've actually done it.

---

## Exercise 5: Mutate a Pod — add a default label automatically

**Goal:** See mutation fix a workload instead of blocking it.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: add-default-label }
spec:
  rules:
    - name: add-team-label
      match: { any: [{ resources: { kinds: [Pod] } }] }
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              team: unassigned
EOF
kubectl run mutated --image=nginx:1.25
kubectl get pod mutated -o jsonpath='{.metadata.labels}'
```

**Expected output:** The Pod carries `team=unassigned` even though you never set it.

**Why it matters:** Mutation is the "fix it for you" half of Kyverno — the answer to toil that validate alone can't remove.

---

## Exercise 6: Write and unit-test a Rego policy with OPA

**Goal:** Test policy logic before it ever reaches a cluster.

**Commands:**

```bash
cat > policy.rego <<'EOF'
package kubernetes.admission

deny[msg] {
  container := input.request.object.spec.containers[_]
  not startswith(container.image, "registry.example.com/")
  msg := sprintf("image %v must come from registry.example.com/", [container.image])
}
EOF
cat > policy_test.rego <<'EOF'
package kubernetes.admission

test_denies_external_image {
  deny[_] with input as {"request": {"object": {"spec": {"containers": [{"image": "docker.io/library/nginx:1.25"}]}}}}
}
test_allows_internal_image {
  count(deny) == 0 with input as {"request": {"object": {"spec": {"containers": [{"image": "registry.example.com/nginx:1.25"}]}}}}
}
EOF
opa test policy.rego policy_test.rego -v
```

**Expected output:** Both tests pass — the policy denies external images and allows internal ones.

**Why it matters:** `opa test` is how you prove a policy is correct in CI, before Gatekeeper ever enforces it.

---

## Exercise 7: Install Gatekeeper and enforce the same rule cluster-wide

**Goal:** See the OPA side of the world enforce at admission.

**Commands:**

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper-library/master/library/general/allowedrepos/template.yaml
kubectl apply -f - <<'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata: { name: only-internal-registry }
spec:
  match: { kinds: [{ apiGroups: [""], kinds: [Pod] }] }
  parameters:
    repos: ["registry.example.com/"]
EOF
kubectl run external --image=nginx:1.25
```

**Expected output:** The Pod is rejected by the Gatekeeper constraint, naming the allowed-repos rule.

**Why it matters:** You've now run both engines — Kyverno's YAML and OPA's Rego — and can compare them from experience.

---

## Exercise 8: Add a namespace exception and prove it works

**Goal:** Practice the visible, scoped exception instead of a bypass.

**Commands:**

```bash
kubectl create namespace sandbox
kubectl label namespace sandbox policy-exempt=true
kubectl patch clusterpolicy require-team-label --type=json \
  -p '[{"op":"add","path":"/spec/rules/0/exclude","value":{"any":[{"resources":{"namespaces":["sandbox"]}}]}}'
kubectl run exempt --image=nginx:1.25 -n sandbox
kubectl run still-blocked --image=nginx:1.25
```

**Expected output:** The sandbox Pod is created; the default-namespace Pod is still rejected.

**Why it matters:** Scoped exceptions are the honest answer to "what if a team genuinely needs an exception" — no secret bypasses.

---

## Exercise 9: Debug a blocked deployment end to end

**Goal:** Build the debugging reflex: message, policy, fix.

**Commands:**

```bash
kubectl run debug-me --image=nginx:1.25 --labels team=platform 2>&1 | tee /tmp/deny.txt
cat /tmp/deny.txt   # note the policy and rule named in the message
kubectl get clusterpolicy
kubectl describe clusterpolicy block-latest-tag
# Fix the workload (pin the version), then re-apply:
kubectl run debug-me --image=nginx:1.25 --labels team=platform
kubectl get pod debug-me
```

**Expected output:** The first attempt fails with a named policy; after pinning the version, the Pod is created.

**Why it matters:** This is the full loop you describe in Q12 — read the denial, read the policy, fix the workload or the policy in Git.

