# Services, Ingress and Gateway API Interview Preparation Guide

*How to Answer Kubernetes Networking Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Kubernetes Services](#kubernetes-services)
2. [Ingress](#ingress)
3. [Gateway API](#gateway-api)
4. [Networking Troubleshooting](#networking-troubleshooting)

---

## Kubernetes Services

### Q1: What is a Kubernetes Service and why do Pods need it?

**How to Answer:**

"Pods are ephemeral — they get new IPs every time they restart, so you can't rely on a Pod IP for anything. A Service gives you one stable virtual IP and DNS name that fronts a set of Pods selected by labels."

"It sits in front of the Pods and load-balances traffic across all the healthy ones. Without Services, every Pod restart would break anything trying to reach it."

**Key Point:** "A Service is the stable front door — one IP and DNS name for a constantly changing set of Pods."

---

### Q2: What are the Service types and when do you use each one?

**How to Answer:**

"ClusterIP is the default — internal-only, and honestly that's what 90% of my Services are, because most traffic is service-to-service. NodePort opens a fixed port on every node, which is handy for quick debugging but I wouldn't ship it to production."

"LoadBalancer asks your cloud provider to provision an actual load balancer in front of the Service — that's the standard way to expose something on AWS or GCP. ExternalName is the odd one out: it doesn't proxy traffic at all, it just returns a CNAME, so it's for pointing at things outside the cluster like an RDS endpoint."

**Key Point:** "ClusterIP for internal, LoadBalancer for public cloud exposure, NodePort for debugging, ExternalName for DNS aliases to external systems."

---

### Q3: What is a headless Service and where does it actually matter?

**How to Answer:**

"You set `clusterIP: None` and Kubernetes skips the virtual IP entirely — DNS queries return the individual Pod IPs instead of one Service IP. That's the whole trick."

"I reach for it with stateful stuff like databases or Kafka, where each replica is distinct and the client needs to talk to a specific Pod, not a random one behind a load balancer. Combined with a StatefulSet, each Pod also gets a stable DNS name like `db-0.db-headless.default.svc`."

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: postgres
```

**Key Point:** "Headless Services skip load balancing so clients can discover and address each Pod individually — the standard choice for StatefulSets."

---

### Q4: How does kube-proxy actually route traffic to Pods?

**How to Answer:**

"kube-proxy watches the API server for Service and Endpoints changes, and on every node it programs the routing — these days usually iptables or IPVS rules, sometimes eBPF. When a packet hits the Service's virtual IP, those rules DNAT it to one of the healthy Pod IPs."

"The Service IP itself doesn't exist on any interface — it's purely a routing fiction that every node's rules agree on. The one interview trap here: if a Pod isn't in the Endpoints object, it gets zero traffic, no matter what the Service selector says."

**Key Point:** "kube-proxy turns the fake Service IP into real Pod IPs using per-node iptables/IPVS rules driven by the Endpoints object."

---

### Q5: How does DNS work for Services inside the cluster?

**How to Answer:**

"CoreDNS runs in the cluster and serves the `cluster.local` zone — every Service automatically gets a DNS name like `payments.default.svc.cluster.local`, and short names like `payments` resolve within the same namespace. I rely on this constantly instead of hardcoding IPs."

"For headless Services it returns A records for each Pod IP, and for ExternalName Services it returns the CNAME. The trap interviewers check: if DNS resolution works but traffic fails, the problem is your Service selectors or NetworkPolicies, not DNS."

**Key Point:** "CoreDNS gives every Service a predictable DNS name — `<svc>.<namespace>.svc.cluster.local` — so nothing in the cluster should ever hardcode a Pod IP."

---

## Ingress

### Q6: What problem does Ingress solve that Services can't?

**How to Answer:**

"A LoadBalancer Service gets you one public endpoint per Service, which means one cloud load balancer per app — expensive and messy at scale. Ingress gives you one entry point that routes by hostname and URL path, so `api.example.com` and `app.example.com` can share a single load balancer."

"It's the standard way to expose HTTP and HTTPS apps. Anything with host-based or path-based routing — basically every web app — goes through Ingress rather than a pile of LoadBalancer Services."

**Key Point:** "Ingress is one shared entry point with host and path routing — the cheap, sane way to expose many HTTP apps instead of one load balancer per Service."

---

### Q7: What does an Ingress spec look like in practice?

**How to Answer:**

"You define rules: which host, which path, and which backend Service and port each one hits. The `ingressClassName` picks which controller actually implements it, because the resource alone does nothing."

"TLS goes in a `tls` block referencing a Secret with the cert — and a classic gotcha is that the Secret must live in the same namespace as the Ingress. I also always remember that path matching needs `pathType`, usually Prefix or Exact."

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  tls:
  - hosts: [app.example.com]
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port: {number: 80}
```

**Key Point:** "An Ingress is just routing intent — hosts, paths, TLS Secret, and the class that implements it — the controller does the real work."

---

### Q8: How does an Ingress controller actually work end to end?

**How to Answer:**

"The controller — NGINX, Traefik, whatever — watches Ingress resources and translates them into its own config, like generating an nginx.conf and reloading it. Traffic hits the controller's own Service, which is usually a LoadBalancer type, and then it reverse-proxies to your backend Services based on the rules."

"The mental model interviewers want: Ingress is the API object, the controller is the implementation. Deploying an Ingress YAML without a controller installed does absolutely nothing, and that's the number one 'why isn't my Ingress working' answer."

**Key Point:** "Ingress is the rulebook; the controller is the reader — it watches Ingress objects and configures a real reverse proxy to enforce them."

---

### Q9: What are the classic Ingress mistakes interviewers love to ask about?

**How to Answer:**

"Forgetting `ingressClassName` so no controller picks it up — or having two controllers fight over it. Pointing the backend at a Service port that doesn't exist, or a selector that matches no Pods, so you get the default backend 404."

"TLS misconfig is the other classic: the Secret in the wrong namespace, or cert-manager annotations missing so the cert never gets issued. And on managed clusters, people forget the controller itself needs a LoadBalancer Service or there's no public IP at all."

**Key Point:** "Most broken Ingresses are one of four things: no controller, wrong class, backend pointing at nothing, or the TLS Secret in the wrong place."

---

## Gateway API

### Q10: What is the Gateway API and how is it different from Ingress?

**How to Answer:**

"The Gateway API is the next-generation replacement for Ingress — it's a set of CRDs that fixes Ingress's biggest weaknesses: everything is annotation-driven, there's only one rewrite model, and there's no clean way to share routing across teams."

"The big philosophical shift is role separation: infra teams own the Gateway, app teams own their Routes. With Ingress, everyone fights over the same annotations on one object. Gateway API is also designed for more than HTTP from day one — TCP, UDP, TLS, and mesh use cases are first-class."

**Key Point:** "Gateway API replaces annotation soup with structured, role-separated routing — infra owns the Gateway, app teams attach their own Routes."

---

### Q11: How do GatewayClass, Gateway and HTTPRoute split responsibilities?

**How to Answer:**

"GatewayClass is the template — it says which controller implementation backs things, like 'istio' or the cloud provider's Gateway. The Gateway is the actual listener config: which ports, which hostnames, what TLS certificates — that's owned by the platform or infra team."

"HTTPRoute is where app teams live: it attaches to a Gateway and declares path and header matching plus which backend Service gets the traffic. So infra provisions one Gateway per environment, and each team self-serves their own routing without touching anyone else's config."

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: prod-gateway
  rules:
  - matches:
    - path: {type: PathPrefix, value: /api}
    backendRefs:
    - name: api-svc
      port: 80
```

**Key Point:** "GatewayClass picks the implementation, Gateway owns the listeners, HTTPRoute owns the app's routing — clean separation of concerns."

---

### Q12: Would you still use Ingress on a new cluster today?

**How to Answer:**

"Honestly, for a greenfield cluster I'd reach for the Gateway API — it's GA, the major controllers support it, and the role separation saves real pain as teams grow. Ingress is effectively frozen: no major new features are landing."

"But I'd still be pragmatic — if the team already knows Ingress, the controllers are running fine, and the routing needs are simple, ripping it out buys you nothing. The interview-safe answer is: Gateway API for new builds, Ingress is fine to keep where it already works."

**Key Point:** "Gateway API for greenfield, keep Ingress where it already works — it's stable, just not evolving."

---

## Networking Troubleshooting

### Q13: Your Service returns connection refused — how do you debug it?

**How to Answer:**

"I start at the Endpoints, not the Service — `kubectl get endpoints <svc>` tells me immediately whether any Pod is actually registered. Empty Endpoints means the selector doesn't match, which is the most common cause by far."

"If Endpoints are populated, I check the Pod itself: is the container actually listening on that port, did the readiness probe fail, is the app crashing? Then I `kubectl port-forward` to the Pod to test it directly — if that works but the Service doesn't, the problem is in the Service or kube-proxy layer."

**Key Point:** "Check Endpoints first — empty Endpoints means a selector mismatch; populated Endpoints means the problem is in the Pod or the proxy layer."

---

### Q14: Your Ingress shows a 404 or the default backend — what's the checklist?

**How to Answer:**

"Default backend means the controller got the request but matched no rule, so I re-read my rules: host spelling, path and pathType, and whether the request actually hits the right hostname — people test with curl and forget the Host header."

"Then I verify the backend Service exists with healthy Endpoints, and that the Ingress has the right `ingressClassName` for the controller that's actually installed. If TLS is involved, I check the Secret exists in the same namespace — that's a silent killer."

**Key Point:** "A default-backend 404 is a routing miss — check host, path, class, backend Endpoints, then the TLS Secret."

---

### Q15: Traffic is uneven across Pods or sessions keep dropping — what do you check?

**How to Answer:**

"First suspect is usually `sessionAffinity: ClientIP` someone set for sticky sessions, or long-lived connections — kube-proxy balances connections, not requests, so one gRPC connection pins to one Pod forever. That's a classic 'my load balancer isn't balancing' interview trap."

"For session drops I check `externalTrafficPolicy`: the default Cluster mode SNATs everything so the Pod never sees the real client IP, which breaks IP-based rate limiting and session affinity. Switching to Local preserves the client IP but only routes to nodes that actually run the Pod."

**Key Point:** "kube-proxy balances connections, not requests — and SNAT hides client IPs unless you set externalTrafficPolicy to Local."
