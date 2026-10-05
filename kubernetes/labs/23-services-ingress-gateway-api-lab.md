# Services, Ingress & Gateway API — Hands-On Lab

Practical exercises for topic 23. You need a working cluster (minikube, kind, or any cloud cluster) and `kubectl` configured. For exercises 7–8 you'll need an Ingress controller (minikube: `minikube addons enable ingress`).

---

## Exercise 1: Expose a Deployment with a ClusterIP Service

**Goal:** Create a Service and watch kube-proxy register Endpoints.

**Commands:**

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=2
kubectl expose deployment web --port=80
kubectl get svc web
kubectl get endpoints web
```

**Expected output:** A ClusterIP Service with a virtual IP like `10.96.x.x`, and an Endpoints object listing both Pod IPs on port 80.

**Why it matters:** Shows the Service → Endpoints → Pods chain — everything about debugging Services starts here.

---

## Exercise 2: Break the selector on purpose

**Goal:** See what an empty Endpoints object looks like.

**Commands:**

```bash
kubectl create service clusterip broken --tcp=80:80
kubectl get endpoints broken
# now fix it by editing the selector to match your web deployment
kubectl patch svc broken -p '{"spec":{"selector":{"app":"web"}}}'
kubectl get endpoints broken
```

**Expected output:** `broken` initially has `<none>` as Endpoints. After patching the selector, the Pod IPs appear.

**Why it matters:** Selector mismatch is the #1 reason Services "don't work" — this is the exact symptom and fix.

---

## Exercise 3: NodePort and accessing from outside

**Goal:** Understand how NodePort exposes traffic on every node.

**Commands:**

```bash
kubectl expose deployment web --port=80 --type=NodePort --name=web-np
kubectl get svc web-np
curl http://localhost:$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}') -s -o /dev/null -w "%{http_code}\n"
```

**Expected output:** A Service with a port in the 30000–32767 range, and curl returning `200`.

**Why it matters:** NodePort is the quick-and-dirty debugging path — useful to know, but you see why nobody ships it.

---

## Exercise 4: Headless Service and Pod DNS

**Goal:** See DNS return individual Pod IPs instead of one virtual IP.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
EOF
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup web-headless
```

**Expected output:** nslookup returns two A records — the individual Pod IPs, not a Service IP.

**Why it matters:** This is how StatefulSets give each replica its own address — the basis for clustered databases on Kubernetes.

---

## Exercise 5: CoreDNS and the full Service FQDN

**Goal:** Resolve a Service by its full DNS name from inside a Pod.

**Commands:**

```bash
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup web.default.svc.cluster.local
kubectl run dns-test2 --image=busybox:1.36 --rm -it --restart=Never -- nslookup web
```

**Expected output:** Both resolve to the same ClusterIP — proving short names work within a namespace and the FQDN pattern is predictable.

**Why it matters:** Service discovery is just DNS — once you know the FQDN pattern you can construct it for any Service, any namespace.

---

## Exercise 6: ExternalName for an outside dependency

**Goal:** Alias an external host without any proxying.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.internal.example.com
EOF
kubectl get svc external-db
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup external-db
```

**Expected output:** nslookup returns a CNAME to `db.internal.example.com` — no ClusterIP, no Endpoints.

**Why it matters:** Lets you point workloads at external systems (RDS, SaaS) through the same DNS naming pattern as internal Services.

---

## Exercise 7: First Ingress with path routing

**Goal:** Route two paths to two Services through one Ingress.

**Commands:**

```bash
kubectl create deployment api --image=nginx:1.25
kubectl expose deployment api --port=80
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend: {service: {name: web, port: {number: 80}}}
      - path: /api
        pathType: Prefix
        backend: {service: {name: api, port: {number: 80}}}
EOF
kubectl get ingress demo
```

**Expected output:** The Ingress gets an ADDRESS (on minikube it may stay pending — check `minikube tunnel` or use NodePort). Both paths route to their backends through one entry point.

**Why it matters:** One Ingress replacing multiple LoadBalancers is the core cost/ops argument for path-based routing.

---

## Exercise 8: Watch the controller translate Ingress to config

**Goal:** See the Ingress → controller → proxy config pipeline.

**Commands:**

```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=20 | grep -i "reload\|sync"
# describe shows the controller's view of your Ingress
kubectl describe ingress demo
```

**Expected output:** Controller logs showing config reloads triggered by your Ingress; describe shows the resolved rules and backends.

**Why it matters:** Makes concrete that Ingress is just intent — the controller does the real proxy configuration.

---

## Exercise 9: Gateway API — HTTPRoute basics (optional)

**Goal:** Do the same path routing with the Gateway API.

**Commands:**

```bash
# requires a Gateway API-capable controller, e.g. kind with the standard channel:
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

**Expected output:** GatewayClass, Gateway, HTTPRoute CRDs installed — ready for a controller to implement.

**Why it matters:** Shows the CRD-based model: structured resources instead of annotations, and the infra/app split in action.

---

## Exercise 10: externalTrafficPolicy and client IPs

**Goal:** See SNAT hide the client IP, then preserve it.

**Commands:**

```bash
kubectl patch svc web-np -p '{"spec":{"externalTrafficPolicy":"Local"}}'
kubectl get svc web-np -o jsonpath='{.spec.externalTrafficPolicy}'
# restore afterwards:
kubectl patch svc web-np -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

**Expected output:** Policy flips to `Local` — now only nodes running Pods receive traffic, and the app sees real client IPs.

**Why it matters:** The difference between working rate limiting and broken IP-based sessions is this one field.

---

**Cleanup:** `kubectl delete deploy,svc,ingress --all`
