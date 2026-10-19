# Lab: Kubernetes NetworkPolicies

Nine hands-on exercises. You need a cluster with a policy-enforcing CNI — kind
with Calico installed works great and runs locally. Plain kind/Flannel will NOT
enforce policies (exercise 3 makes that visceral).

## Exercise 1: Install a policy-enforcing CNI on kind

**Goal:** Get a cluster where policies actually do something.

**Commands:**

```bash
kind create cluster --name netpol-lab
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n kube-system --timeout=120s
```

**Expected output:** All `calico-node` pods `Ready`. `kubectl get pods -n kube-system` shows Calico components running.

**Why it matters:** This is Q13 made real — without this step, every policy you write for the rest of the lab is a no-op.

## Exercise 2: Prove the default is open

**Goal:** See the flat network before you lock it down.

**Commands:**

```bash
kubectl create ns frontend && kubectl create ns backend
kubectl run web -n frontend --image=nginx --labels=app=web
kubectl run db -n backend --image=nginx --labels=app=db
kubectl wait --for=condition=Ready pod/web -n frontend --timeout=60s
kubectl exec deploy/web -n frontend -- curl -s -o /dev/null -w "%{http_code}\n" \
  http://$(kubectl get pod -n backend -l app=db -o jsonpath='{.items[0].status.podIP}')
```

**Expected output:** `200`. Cross-namespace traffic flows with zero configuration.

**Why it matters:** This is the scary baseline Q3 warns about — two random namespaces, full connectivity, nothing stopping it.

## Exercise 3: Apply default-deny-all and watch it break

**Goal:** Feel isolation kick in.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all, namespace: backend }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
kubectl exec deploy/web -n frontend -- curl -s -m 5 -o /dev/null -w "%{http_code}\n" \
  http://$(kubectl get pod -n backend -l app=db -o jsonpath='{.items[0].status.podIP}')
```

**Expected output:** curl times out (no response code, exit 28). Run the same command from a pod *inside* backend — it fails too, because egress is denied as well.

**Why it matters:** Empty podSelector + both policyTypes is the master switch. Everything you build from here is opening deliberate holes in this wall.

## Exercise 4: Open one hole — frontend to backend port 80

**Goal:** Write your first real allow rule (Q8 pattern).

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: allow-web-to-db, namespace: backend }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: frontend } }
          podSelector: { matchLabels: { app: web } }
      ports: [{ port: 80 }]
EOF
kubectl exec deploy/web -n frontend -- curl -s -o /dev/null -w "%{http_code}\n" \
  http://$(kubectl get pod -n backend -l app=db -o jsonpath='{.items[0].status.podIP}')
```

**Expected output:** `200` again — the curl pod matches both selectors, so the single rule opens the path.

**Why it matters:** This is the allow-list mindset in action: one caller, one port, everything else still dropped.

## Exercise 5: Watch an unauthorized pod get dropped

**Goal:** Prove the union-isolation works both ways.

**Commands:**

```bash
kubectl run attacker -n frontend --image=busybox -- sleep 3600
kubectl exec attacker -n frontend -- wget -q -T 5 -O /dev/null \
  http://$(kubectl get pod -n backend -l app=db -o jsonpath='{.items[0].status.podIP}') \
  && echo ALLOWED || echo DENIED
```

**Expected output:** `DENIED` (wget exits non-zero on timeout). The attacker pod doesn't carry the `app=web` label, so no rule matches it.

**Why it matters:** Label-based selection is the whole game — change a pod's labels and its network posture changes with it. Audit labels like firewall rules.

## Exercise 6: Trigger the DNS footgun on purpose

**Goal:** Experience Q11's most famous gotcha firsthand.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-egress-broken, namespace: backend }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Egress]
  egress:
    - to: [{ podSelector: { matchLabels: { app: web } } }]
      ports: [{ port: 80 }]
EOF
kubectl exec deploy/db -n backend -- nslookup kubernetes.default
```

**Expected output:** `nslookup` fails with a timeout — the db pod can't reach CoreDNS in kube-system because egress only allows port 80 to the web pods.

**Why it matters:** Nobody remembers DNS is network traffic until it breaks. This is why every egress policy in production needs the port-53 rule to CoreDNS.

## Exercise 7: Fix the egress policy with a DNS exception

**Goal:** Write the correct two-rule egress pattern.

**Commands:**

```bash
kubectl delete networkpolicy db-egress-broken -n backend
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: db-egress-fixed, namespace: backend }
spec:
  podSelector: { matchLabels: { app: db } }
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports:
        - { port: 53, protocol: UDP }
        - { port: 53, protocol: TCP }
    - to: [{ podSelector: { matchLabels: { app: web } } }]
      ports: [{ port: 80 }]
EOF
kubectl exec deploy/db -n backend -- nslookup kubernetes.default
```

**Expected output:** `nslookup` succeeds, resolving `kubernetes.default`. Delete the policy when done: `kubectl delete networkpolicy db-egress-fixed -n backend`.

**Why it matters:** This two-rule shape (DNS exception + real destinations) is the template you'll copy for every egress policy you ever write.

## Exercise 8: Inspect what a policy actually selected

**Goal:** Learn the debug command from Q14.

**Commands:**

```bash
kubectl describe networkpolicy allow-web-to-db -n backend
kubectl get networkpolicy -n backend -o wide
```

**Expected output:** `describe` shows the podSelector, ingress rules, and the matching pods. If it shows zero matched pods, your labels don't line up — the #1 policy bug.

**Why it matters:** Most "my policy doesn't work" cases are label typos. This is the 10-second check that catches them.

## Exercise 9: Simulate the ping test from the interview answer

**Goal:** Reproduce the Q13 enforcement check on your own cluster.

**Commands:**

```bash
kubectl run probe-a --image=busybox -n frontend -- sleep 3600
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: deny-all, namespace: frontend }
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
kubectl exec probe-a -n frontend -- ping -c 2 -W 2 \
  $(kubectl get pod web -n frontend -o jsonpath='{.status.podIP}') && echo PING_OK || echo PING_BLOCKED
```

**Expected output:** `PING_BLOCKED`. Same-namespace ping fails under deny-all — proof the CNI is enforcing. (Clean up: `kind delete cluster --name netpol-lab`.)
