# Kubernetes Networking Deep Dive — Hands-On Lab

Nine drills that turn Kubernetes networking answers into real experience. Each exercise gives you a goal, the steps, what you should see, and why it matters in the interview.

---

## Exercise 1: Trace a packet from pod to pod across two nodes

**Goal:** A written map of the exact path a packet takes between pods on different nodes — and which component owns each hop.

**Steps:**

```bash
# Pick a cluster (kind, minikube, or a cloud cluster you can run pods in).
# Find two pods on different nodes, then trace the path between them.
kubectl get pods -o wide --all-namespaces | head -20
# Note the pod IPs and their nodes. Then check which CNI is installed:
kubectl get pods -n kube-system | grep -E "calico|cilium|flannel|weave"
# Inside one pod, ping the other pod's IP and record the round trip:
kubectl run -it --rm net-trace --image=nicolaka/netshoot --restart=Never -- \
  ping -c 3 <OTHER_POD_IP>
```

**Expected output:** One page naming each hop — pod veth interface, the node's bridge or routing table, the overlay (VXLAN) or BGP route to the other node, and the destination pod — with the CNI in use and how you confirmed it.

**Why it matters:** This is Q1 and Q2 made concrete — "the flat pod network is a promise the CNI keeps" only counts when you can name the mechanism keeping it on your cluster.

---

## Exercise 2: Compare CNI behaviour — overlay vs BGP vs eBPF

**Goal:** A one-page comparison of what changes when the CNI changes, based on something you actually ran.

**Steps:**

```bash
# On a kind cluster, check the default CNI (kindnet) and its limits:
kubectl get pods -n kube-system
kubectl get networkpolicy -A   # kindnet does not enforce these — that is the point
# Now create a second kind cluster with a real CNI (Calico or Cilium via their quickstart),
# apply a default-deny policy in a test namespace, and watch the traffic stop:
kubectl create namespace cni-test
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: cni-test
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

**Expected output:** A short table — CNI, encapsulation or routing method, policy enforcement (yes/no), and one observed difference (e.g. traffic blocked on Calico but not on kindnet).

**Why it matters:** This is Q2 and Q10 in one drill — a NetworkPolicy is a decoration until the CNI enforces it, and interviewers who run clusters know which CNI does what.

---

## Exercise 3: See kube-proxy's rules for a real Service

**Goal:** Proof that a Service is just rules on every node, read from the node itself.

**Steps:**

```bash
# Create a Service with two backend pods, then read the rules kube-proxy wrote:
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80
kubectl get endpoints web
# Read the iptables rules (or IPVS table, depending on the cluster's mode):
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o jsonpath='{.items[0].spec.containers[0].command}'
iptables-save | grep -i web | head -10
# Delete one pod and watch the endpoints — and the rules — update:
kubectl delete pod -l app=web --force --grace-period=0
kubectl get endpoints web
```

**Expected output:** The Service's ClusterIP, its endpoints before and after the delete, and the iptables or IPVS lines that rewrite the ClusterIP to pod IPs.

**Why it matters:** This is Q3 made concrete — "a Service is a stable name over rules on every node" is an answer you can defend once you've read the rules yourself.

---

## Exercise 4: Pick the right Service type for three real cases

**Goal:** A written decision for three services, with the type you'd choose and the reason — no hedging.

**Steps:**

```bash
# For each case, write the type and one line of justification:
# 1. A Postgres database used only by pods in the same cluster.
# 2. A public marketing site that needs a real external IP and TLS.
# 3. A debug dashboard you want to reach from your laptop for an afternoon.
# Then verify case 2's shape on a cloud cluster or with a local load balancer (e.g. MetalLB on kind):
kubectl get svc -A | grep -E "LoadBalancer|NodePort"
# Check what a NodePort actually opens on the node:
ss -tlnp | grep 3xxxx   # the NodePort range is 30000-32767
```

**Expected output:** Three decisions — ClusterIP for the database, LoadBalancer for the public site, NodePort (or port-forward) for the debug dashboard — each with one honest reason and one thing you'd never do with that type.

**Why it matters:** This is Q4 in interview form — the types are a ladder of exposure, and the interviewer is checking you don't reach for NodePort in production.

---

## Exercise 5: Install an ingress controller and route two hosts

**Goal:** Two hostnames routing to two different Services through one controller — the whole flow, working.

**Steps:**

```bash
# Install NGINX Ingress on a local cluster, then create two Services and one Ingress:
kubectl create deployment shop --image=nginx --replicas=1
kubectl create deployment api --image=httpd --replicas=1
kubectl expose deployment shop --port=80
kubectl expose deployment api --port=80
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: two-hosts
spec:
  ingressClassName: nginx
  rules:
    - host: shop.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: shop, port: {number: 80}}
    - host: api.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: api, port: {number: 80}}
EOF
# Point the hosts at the controller and curl both:
curl -H "Host: shop.test" http://<INGRESS_IP>/ | head -3
curl -H "Host: api.test" http://<INGRESS_IP>/ | head -3
```

**Expected output:** Both curls returning different backends through the same controller IP, plus the answer to: what happens if you delete the controller deployment?

**Why it matters:** This is Q5 made real — Ingress is the rules, the controller is the router, and you feel the difference the moment the controller disappears.

---

## Exercise 6: Split traffic with Gateway API — canary without annotations

**Goal:** A canary split done the Gateway API way, with the weight change as the rollback.

**Steps:**

```bash
# With Gateway API CRDs installed, create a Gateway and an HTTPRoute with two backends:
kubectl apply -f - <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-canary
spec:
  parentRefs:
    - name: main-gateway
  hostnames: ["shop.example.com"]
  rules:
    - matches:
        - path: {type: PathPrefix, value: /}
      backendRefs:
        - name: shop-stable
          port: 80
          weight: 90
        - name: shop-canary
          port: 80
          weight: 10
EOF
# Send 20 requests and count how many hit the canary (use a header or a version string in the response):
for i in $(seq 1 20); do curl -s http://shop.example.com/version; done | sort | uniq -c
# Now roll back by setting the canary weight to 0 and re-apply — time how long the flip takes.
```

**Expected output:** Roughly 2 of 20 requests hitting the canary, then 0 of 20 after the weight change, with the time the rollback took written down.

**Why it matters:** This is Q6 and Q8 in one drill — the split lives in the route, not in annotations, and the rollback is a weight change, not a deploy.

---

## Exercise 7: Automate TLS with cert-manager and break it on purpose

**Goal:** A working HTTPS endpoint with an auto-renewed cert — plus the failure mode you found by breaking it.

**Steps:**

```bash
# Install cert-manager, create a ClusterIssuer (Let's Encrypt staging for a test domain,
# or a self-signed issuer for a local cluster), and annotate an Ingress to get a cert:
kubectl get certificate -A
kubectl get secret <your-tls-secret> -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -dates -subject
# Now break it on purpose: move the Secret to another namespace and watch the Ingress fail,
# then check what cert-manager reports:
kubectl describe certificate <your-cert> | grep -A 5 Events
```

**Expected output:** The cert's real expiry date, the exact error when the Secret is in the wrong namespace, and one line on how you'd get paged before a cert expires in production.

**Why it matters:** This is Q7 made concrete — certs are a Secret next to the Ingress, and the wrong-namespace failure is the one that bites at 2 AM.

---

## Exercise 8: Write a default-deny policy — and catch the DNS mistake yourself

**Goal:** A locked-down namespace where apps still work, because you remembered DNS.

**Steps:**

```bash
# Create a namespace, deploy an app that calls an external API, then apply default-deny:
kubectl create namespace locked
kubectl -n locked create deployment app --image=curlimages/curl -- sleep infinity
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: locked
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
EOF
# Try a DNS lookup from the pod — it should fail. Then add the DNS egress rule
# (UDP/TCP 53 to kube-dns in kube-system) and watch it succeed:
kubectl -n locked exec deploy/app -- nslookup kubernetes.default
```

**Expected output:** The failed lookup before the DNS rule, the successful one after, and the final policy YAML with the DNS allowance in it.

**Why it matters:** This is Q10's classic mistake, felt firsthand — a deny-all that blocks UDP 53 breaks name resolution, and it looks exactly like a network failure.

---

## Exercise 9: Debug a "pod can't reach the Service" outage, end to end

**Goal:** A timed run of the five-check debug chain on a broken setup you built yourself.

**Steps:**

```bash
# Break something on purpose — pick one: delete the endpoints' pods, add a wrong label
# to the Service selector, or apply a deny policy — then run the chain:
kubectl run -it --rm net-debug --image=nicolaka/netshoot --restart=Never -- bash
# 1. nslookup <svc>.<ns>.svc.cluster.local
# 2. curl -v http://<svc>.<ns>:<port>/healthz
# 3. curl the pod IP directly
# 4. kubectl get networkpolicy -n <ns>
# 5. curl the container port from inside the pod; check container logs
# Write down which check found it and how long the whole run took.
```

**Expected output:** The broken link you planted, the check that caught it, and the total time — plus one sentence on which check you'd run first next time and why.

**Why it matters:** This is Q11 as a skill, not an answer — DNS, endpoints, pod-to-pod, policy, app, in that order, under time pressure, is what the interviewer is really testing.
