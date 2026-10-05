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
