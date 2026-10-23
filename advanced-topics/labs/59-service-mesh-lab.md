# Service Mesh Deep Dive — Hands-On Lab

Nine drills that make mesh answers sound like real experience. Each exercise gives you a goal, the commands, what you should see, and why it matters in the interview.

---

## Exercise 1: Install Istio in demo mode on a local cluster

**Goal:** Get a working control plane in minutes.

**Commands:**

```bash
istioctl install --set profile=demo -y
kubectl get pods -n istio-system
kubectl label namespace default istio-injection=enabled
```

**Expected output:** `istiod` and the ingress gateway are Running in `istio-system`.

**Why it matters:** Interviewers trust answers from someone who has actually seen istiod come up — and seen it fail.

---

## Exercise 2: Prove sidecar injection happened

**Goal:** Verify a pod really has the proxy.

**Commands:**

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[*].name}{"\n"}{end}'
```

**Expected output:** Every Bookinfo pod lists two containers — the app and `istio-proxy`.

**Why it matters:** "2/2 Ready" is the fastest injection check, and the first thing you check in a real incident too.

---

## Exercise 3: Open the Bookinfo app through the gateway

**Goal:** See north-south traffic enter the mesh.

**Commands:**

```bash
kubectl get svc istio-ingressgateway -n istio-system
export GATEWAY_URL=<external-ip>:<port>
curl -s http://$GATEWAY_URL/productpage | head
```

**Expected output:** The Bookinfo product page HTML returns with a 200.

**Why it matters:** It locks in the gateway-at-the-edge, mesh-inside picture you draw in interviews.

---

## Exercise 4: Split traffic 95/5 between reviews v1 and v3

**Goal:** Run a real weight-based canary.

**Commands:**

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-95-05.yaml 2>/dev/null || kubectl apply -f samples/bookinfo/networking/destination-rule-reviews.yaml
istioctl proxy-config routes deploy/productpage-v1 | head -20
# then apply a VirtualService with weight 95 (v1) / 5 (v3) and curl the page repeatedly
for i in $(seq 1 20); do curl -s http://$GATEWAY_URL/productpage > /dev/null; echo ok $i; done
```

**Expected output:** Most product pages show no stars (v1), a few show red stars (v3) — roughly 1 in 20.

**Why it matters:** Canary-by-weight stops being theory once you have watched the split actually happen.

---

## Exercise 5: Add a timeout and a careful retry

**Goal:** Feel policy-driven resilience.

**Commands:**

```bash
cat > /tmp/reviews-vs.yaml <<'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: reviews }
spec:
  hosts: [reviews]
  http:
  - route: [{ destination: { host: reviews, subset: v1 } }]
    timeout: 3s
    retries: { attempts: 2, perTryTimeout: 1s, retryOn: 5xx }
EOF
kubectl apply -f /tmp/reviews-vs.yaml
```

**Expected output:** `kubectl get virtualservice reviews` shows the new timeout/retry policy active.

**Why it matters:** You can now say exactly which calls you would retry (idempotent, 5xx only) and why — the follow-up everyone gets.

---

## Exercise 6: Turn on STRICT mTLS and prove it

**Goal:** Enforce encrypted service-to-service traffic.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default, namespace: istio-system }
spec: { mtls: { mode: STRICT } }
EOF
istioctl x describe pod <a-bookinfo-pod>
```

**Expected output:** The describe output shows PeerAuthentication STRICT and TLS connections between proxies.

**Why it matters:** "PERMISSIVE first, then STRICT" is a migration story — running it once makes the story credible.

---

## Exercise 7: Deny a caller with an AuthorizationPolicy

**Goal:** See identity-based access control work.

**Commands:**

```bash
# deny productpage -> reviews, watch the page lose its reviews section, then delete the policy
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata: { name: deny-reviews, namespace: default }
spec:
  selector: { matchLabels: { app: reviews } }
  action: DENY
  rules: [{ from: [{ source: { principals: ["cluster.local/ns/default/sa/bookinfo-productpage"] } }] }]
EOF
```

**Expected output:** The product page loads but reviews fail; deleting the policy restores them.

**Why it matters:** It proves authorization is about cryptographic identity, not IPs — a line that lands in security follow-ups.

---

## Exercise 8: Read the mesh in Kiali / Grafana

**Goal:** Connect telemetry to a topology you can describe.

**Commands:**

```bash
istioctl dashboard kiali
# generate traffic first: for i in $(seq 1 100); do curl -s http://$GATEWAY_URL/productpage > /dev/null; done
kubectl -n istio-system get svc prometheus grafana
```

**Expected output:** Kiali shows the Bookinfo graph with request rates and error percentages per edge.

**Why it matters:** "The mesh gives per-route metrics for free" is only convincing if you have read them off a real graph.

---

## Exercise 9: Break it — mTLS mismatch drill

**Goal:** Practice the classic mesh debugging path.

**Commands:**

```bash
# set one namespace STRICT while a non-mesh caller still sends plain HTTP, reproduce the 503s
kubectl logs <caller-pod> -c istio-proxy --tail=20
istioctl analyze -n default
# fix: align PeerAuthentication modes, re-test with curl
```

**Expected output:** Proxy logs show connection failures until the modes align; `istioctl analyze` reports no errors after the fix.

**Why it matters:** Half of mesh interviews end at troubleshooting — a rehearsed mismatch drill is a ready-made answer.

---
