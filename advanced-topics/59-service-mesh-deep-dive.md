# Service Mesh Deep Dive Interview Preparation Guide

*How to Answer Service Mesh Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Why a Service Mesh](#why-a-service-mesh)
2. [How It Works](#how-it-works)
3. [Traffic Management](#traffic-management)
4. [Security and mTLS](#security-and-mtls)
5. [Observability and Troubleshooting](#observability-and-troubleshooting)

---

## Why a Service Mesh

### Q1: What is a service mesh, in one simple answer?

**How to Answer:**

"I think of it as infrastructure for service-to-service calls. Instead of every app re-implementing retries, timeouts, TLS and metrics, a sidecar proxy sits next to each service and handles all of that. The app just makes a local call, the proxy does the hard part. Istio and Linkerd are the two names I know best."

**Key Point:** "A mesh moves retries, TLS, routing and telemetry out of app code into a sidecar proxy beside every service."

---

### Q2: When do you actually need a mesh, and when is it overkill?

**How to Answer:**

"I reach for it when I have many services, multiple teams, and I need the same traffic and security policy everywhere. If I only have three services, a good library and an ingress are honestly enough. A mesh adds CPU, latency and a whole control plane to operate. I always name that cost before I recommend it."

**Key Point:** "Many services + shared policy = mesh. A handful of services = probably overkill."

---

### Q3: Service mesh vs API gateway vs ingress — how do you separate them?

**How to Answer:**

"I separate them by direction. Ingress and gateways handle north-south traffic — users coming into the cluster. A mesh handles east-west traffic — services talking to each other inside. They complement each other, they don't replace each other. In interviews I draw the gateway at the edge and the mesh behind it."

**Key Point:** "Gateway/ingress = north-south at the edge. Mesh = east-west inside the cluster."

---

## How It Works

### Q4: Explain the data plane and control plane without jargon.

**How to Answer:**

"The data plane is the sidecars — usually Envoy — that actually move traffic and enforce policy on every request. The control plane is the brain that tells those sidecars what the rules are and pushes config when it changes. In Istio that brain is istiod. If the control plane goes down, existing traffic usually keeps flowing, new config just stops updating."

```bash
kubectl get pods -n istio-system   # control plane
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" containers="}{.spec.containers[*].name}{"\n"}{end}'  # spot the istio-proxy sidecar
```

**Key Point:** "Sidecars move traffic, the control plane moves config — and a control-plane outage is not a data-plane outage."

---

### Q5: How does sidecar injection actually work?

**How to Answer:**

"I label a namespace for injection and a mutating webhook adds the proxy container plus an init container that rewrites iptables. After that, all inbound and outbound traffic for the pod flows through Envoy. I verify it by checking the pod has two containers, not one. No app code change is needed, which is the whole selling point."

```bash
kubectl label namespace default istio-injection=enabled
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].name}'  # expect: app istio-proxy
```

**Key Point:** "Label the namespace, the webhook injects Envoy, iptables redirects traffic through it."

---

### Q6: Istio vs Linkerd — how do you choose?

**How to Answer:**

"I choose Linkerd when I want simple and light — it installs fast, uses less resource, and its own proxy is built for Kubernetes only. I choose Istio when I need richer traffic policy, multi-cluster, or VM workloads outside Kubernetes. The tradeoff is power versus operational weight. I say that tradeoff out loud instead of picking a winner."

**Key Point:** "Linkerd = lighter and simpler. Istio = more features, more to operate."

---

## Traffic Management

### Q7: How do you do a canary release with a mesh?

**How to Answer:**

"I use a VirtualService to split traffic by weight — say 95 percent to v1 and 5 percent to v2. I watch error rate and p99 for the canary subset specifically, not the overall average. If it stays clean, I shift weight in steps. No new deployment trick is needed, the split is just routing config."

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata: { name: reviews }
spec:
  hosts: [reviews]
  http:
  - route:
    - destination: { host: reviews, subset: v1 }
      weight: 95
    - destination: { host: reviews, subset: v2 }
      weight: 5
```

**Key Point:** "Weight-based routing in a VirtualService + per-subset metrics = a mesh canary."

---

### Q8: How do retries, timeouts and circuit breaking work in a mesh?

**How to Answer:**

"I set them in routing policy so every service gets the same behaviour. Retries only on idempotent calls, with a small attempt count and backoff — blind retries amplify an outage. Timeouts are per-try and overall, and circuit breaking ejects a host that keeps failing so traffic stops hammering it. These three together are why a mesh helps in incidents."

**Key Point:** "Retry carefully, time out always, eject failing hosts — set once in policy, applied everywhere."

---

### Q9: What is a DestinationRule used for?

**How to Answer:**

"I use it to define what happens after routing picks a service — subsets, load balancing and connection limits. Subsets are how I label v1 and v2 by pod labels so a VirtualService can split between them. It also holds the circuit-breaker thresholds. VirtualService decides where, DestinationRule decides how."

**Key Point:** "VirtualService = where traffic goes. DestinationRule = subsets, balancing and limits for that destination."

---

## Security and mTLS

### Q10: How does mTLS work in a mesh, and what does STRICT mode mean?

**How to Answer:**

"The sidecars get certificates from the control plane and encrypt traffic between services automatically, with identity based on the service account, not the IP. In PERMISSIVE mode the mesh accepts both plain and encrypted traffic, which helps during migration. In STRICT mode it accepts only mTLS. I roll out PERMISSIVE first, prove everything is encrypted, then flip to STRICT."

```bash
kubectl apply -f - <<'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: default, namespace: istio-system }
spec: { mtls: { mode: STRICT } }
EOF
```

**Key Point:** "mTLS is automatic between sidecars; PERMISSIVE migrates, STRICT enforces."

---

### Q11: How do you restrict which services can talk to each other?

**How to Answer:**

"I use authorization policies on top of mTLS identity. Default in a mesh is usually allow, so I move toward deny-by-default for sensitive namespaces and explicitly allow the callers I expect. A policy might say only the frontend service account can call checkout on GET. Because identity is cryptographic, I'm not trusting a spoofable IP or header."

**Key Point:** "mTLS gives identity, AuthorizationPolicy uses it — aim for deny-by-default on sensitive services."

---

## Observability and Troubleshooting

### Q12: A call is failing only inside the mesh. How do you debug it?

**How to Answer:**

"I start with the proxy, not the app. I check the sidecar is injected and ready, then I look at Envoy's config and stats for that pod. I check whether mTLS mode mismatches — one side STRICT, one side plain is a classic. Then I read the access logs on both sidecars: the caller's proxy tells me what it sent, the receiver's tells me what arrived. `istioctl analyze` catches bad config before I dig deeper."

```bash
istioctl analyze -n default
istioctl proxy-config routes <pod> -n default
kubectl logs <pod> -c istio-proxy -n default --tail=50
```

**Key Point:** "Debug the proxies first: injection, config, mTLS mismatch, then both sides' access logs."

---
