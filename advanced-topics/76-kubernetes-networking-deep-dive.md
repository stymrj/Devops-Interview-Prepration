# Kubernetes Networking Deep Dive: CNI, Ingress & Gateway API in Production Interview Preparation Guide

*How to Answer Kubernetes Networking Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [How Packets Actually Move in Kubernetes](#how-packets-actually-move-in-kubernetes)
2. [Ingress & Gateway API: Getting Traffic In](#ingress--gateway-api-getting-traffic-in)
3. [DNS, NetworkPolicy & Debugging in Production](#dns-networkpolicy--debugging-in-production)

---

## How Packets Actually Move in Kubernetes

### Q1: How does a pod get its IP — and how do two pods on different nodes actually talk to each other?

**How to Answer:**

"I'd start with the rule Kubernetes itself enforces: every pod gets its own IP, and any pod can reach any other pod directly, no NAT in between. When a pod is created, the kubelet asks the CNI plugin for an address from the pod CIDR, and that plugin wires up a virtual interface inside the pod's network namespace. Pod-to-pod traffic across nodes works because the CNI builds an overlay or routes between nodes — VXLAN encapsulation in Flannel, BGP routes in Calico — so a packet leaves pod A, gets routed or tunnelled to pod B's node, and arrives at pod B's IP. The interview line: the flat pod network is a promise the CNI keeps, and my job is knowing which mechanism keeps it, because that's what breaks when it doesn't."

**Key Point:** "Every pod gets its own IP and pods reach each other directly with no NAT — the CNI plugin assigns the address and builds the cross-node path, so the flat network is a promise the CNI keeps."

---

### Q2: What is a CNI plugin — and how do Calico, Cilium, and Flannel actually differ?

**How to Answer:**

"I'd say the CNI is just a contract: the kubelet calls a plugin at pod create and delete time, and the plugin sets up the networking. Flannel is the simple one — it hands out IPs and builds a VXLAN overlay between nodes, which is easy to run but has no policy and no observability. Calico adds real routing with BGP instead of an overlay when it can, plus NetworkPolicy enforcement, so it's the workhorse in most production clusters. Cilium replaces kube-proxy with eBPF, so it does service load balancing, network policy, and deep observability in the kernel — it's the most powerful and the hardest to debug when something goes wrong. My interview line: Flannel for learning, Calico for a solid default, Cilium when you want eBPF speed and visibility and you're ready to own it."

**Key Point:** "CNI is the contract the kubelet calls at pod create time — Flannel is a simple VXLAN overlay, Calico adds BGP routing and policy, Cilium does it all in eBPF with the most power and the hardest debugging."

---

### Q3: How does a Service actually work under the hood — what is kube-proxy doing?

**How to Answer:**

"I'd say a Service is a stable virtual IP backed by rules on every node, and kube-proxy is the thing writing those rules. When you create a Service, the control plane assigns a ClusterIP and kube-proxy watches the Endpoints, then programs iptables or IPVS rules on every node so traffic to that ClusterIP gets rewritten to a healthy pod IP. The default iptables mode works but scales badly — every service adds rules to a long chain the kernel walks one by one. IPVS is faster for big clusters because it hashes to the right backend instead of walking a list. And the modern answer is eBPF, where Cilium skips kube-proxy entirely and does the load balancing in the kernel. The interview point: a Service is just a stable name over rules on every node, and the mode you pick decides whether it stays fast at a thousand services."

```bash
# See what kube-proxy actually wrote for a Service — the rules are the truth.
iptables-save | grep -A 5 "my-service" | head -20
kubectl get endpoints my-service
kubectl get svc my-service -o wide
# On a Cilium cluster there is no kube-proxy — check the eBPF maps instead:
kubectl -n kube-system exec ds/cilium -- cilium service list | grep my-service
```

**Key Point:** "A Service is a stable virtual IP backed by rules kube-proxy writes on every node — iptables walks a list, IPVS hashes, eBPF skips kube-proxy entirely, and the mode decides whether it stays fast at a thousand services."

---

### Q4: ClusterIP vs NodePort vs LoadBalancer — when do you actually use each?

**How to Answer:**

"I'd lay it out as a ladder of exposure. ClusterIP is the default and stays inside the cluster — only other pods and services reach it, which is what you want for a database or an internal API. NodePort opens the same port on every node so anything outside can hit node-ip:port — it works everywhere but it's a hack, the port range is limited, and you never expose it directly to users. LoadBalancer asks the cloud provider to provision a real load balancer and point it at the nodes — that's the production path for anything public, and it's the only type that gives you a real external IP. The mistake I call out is exposing things with NodePort in production and then wondering why there's no TLS, no health checking, and no real routing. My line: ClusterIP inside, LoadBalancer outside, NodePort only for debugging and for the ingress controller to sit on."

**Key Point:** "ClusterIP stays inside the cluster, NodePort opens a port on every node and is a debugging hack, LoadBalancer provisions a real cloud load balancer — NodePort in production is how you end up with no TLS and no health checking."

---

## Ingress & Gateway API: Getting Traffic In

### Q5: How does Ingress actually route traffic — and what is the ingress controller doing?

**How to Answer:**

"I'd separate the two things people blur together: the Ingress resource is just a set of rules — host, path, which Service gets the traffic — and the ingress controller is the real process, usually NGINX or Traefik, that reads those rules and configures itself. The flow is simple: a request hits the load balancer, reaches the controller pods, the controller matches the Host header and path against its config, and it forwards to the right Service's ClusterIP. The gotcha is that an Ingress with no controller does nothing — the resource alone is a wish list, and a fresh cluster has no controller until you install one. And the second gotcha: the controller is itself a deployment with a Service, so if its pods are down or its Service is misconfigured, every route is down at once. My line: Ingress is the rules, the controller is the router, and if the controller is down, every route is down."

```yaml
# The rules are declarative — the controller turns them into NGINX config.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api
                port:
                  number: 8080
```

**Key Point:** "Ingress is the routing rules, the controller is the process that reads them and forwards traffic — an Ingress with no controller does nothing, and if the controller is down every route is down."

---

### Q6: Ingress vs Gateway API — why did Kubernetes need a new API at all?

**How to Answer:**

"I'd say Ingress worked, but it was too simple for what teams actually need. It has one resource doing everything — the host, the path, the TLS, the backend — and every controller extended it with its own annotations, so the same YAML means different things on NGINX and Traefik and you can't move between them. Gateway API splits the job into roles: a Gateway is the listener and TLS config that the platform team owns, and an HTTPRoute is the routing rule the app team owns, so teams stop stepping on each other's config. It also has first-class traffic splitting, header matching, and cross-namespace references built in instead of annotation hacks. The interview line: Ingress is one flat rulebook with vendor annotations; Gateway API is a layered API where the platform team owns the front door and app teams own their routes."

**Key Point:** "Ingress is one flat rulebook extended by vendor annotations; Gateway API splits the front door (Gateway, platform-owned) from the routes (HTTPRoute, app-owned), with traffic splitting and header matching built in."

---

### Q7: How do you handle TLS at the ingress — termination, certs, and the mistakes that leak?

**How to Answer:**

"I'd say TLS terminates at the ingress controller, and the whole setup is two pieces: a Secret holding the cert and key, and the Ingress or Gateway referencing it. In practice I never manage certs by hand — cert-manager watches the Ingress, talks to Let's Encrypt, and renews the cert into a Secret before it expires, and it pages me if issuance fails. The mistakes I call out are three: a cert Secret in the wrong namespace so the Ingress can't read it, a wildcard cert shared across teams with no rotation plan, and traffic behind the ingress running plain HTTP because 'it's inside the cluster' — which means anyone who gets a pod in can sniff it. My line: automate the certs with cert-manager, keep the Secret next to the Ingress, and encrypt the hop behind the ingress too if the traffic is sensitive."

```bash
# The two commands that answer most TLS questions at the ingress:
kubectl get certificate -A              # cert-manager status — Ready=True or it isn't
kubectl get secret shop-tls -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -dates -subject  # real expiry date, not the dashboard's guess
curl -vk https://shop.example.com 2>&1 | grep -E "subject|expire"  # what the client actually sees
```

**Key Point:** "TLS terminates at the ingress from a Secret it can read — automate issuance with cert-manager, and don't leave the hop behind the ingress in plain HTTP just because it's inside the cluster."

---

### Q8: How do you run canary and blue-green releases at the ingress layer?

**How to Answer:**

"I'd say the ingress is the cleanest place to split traffic, because it sits in front of everything. With NGINX Ingress you add a second Ingress marked as the canary with a weight annotation — say 10 — and the controller sends one request in ten to the new version while the rest stay on stable. With Gateway API you do the same thing properly: one HTTPRoute with two backends and a weight on each, no annotations needed. The key part is the exit plan — you watch error rate and latency on the canary, and if it goes red you set the weight to zero, which takes seconds and needs no deploy. The trap I call out is splitting traffic without splitting the data — the canary writes to the same database, so a bad migration still hurts everyone. My line: split at the ingress, watch the canary like a hawk, and keep the weight at zero until the data story is safe."

**Key Point:** "Split traffic at the ingress — a canary Ingress with a weight, or two weighted backends in a Gateway API route — watch errors and latency, and roll back by setting the weight to zero in seconds."

---

## DNS, NetworkPolicy & Debugging in Production

### Q9: How does DNS work inside a cluster — and what breaks when CoreDNS goes down?

**How to Answer:**

"I'd say every pod gets a DNS resolver pointed at the kube-dns Service, which is CoreDNS running as a deployment. A name like api.payments.svc.cluster.local resolves inside the cluster because CoreDNS reads the Service and Endpoint objects from the API server — so DNS is only as fresh as CoreDNS's cache and its watch on the API. The failure I see most is CoreDNS itself: too few replicas on a big cluster, or a bad Corefile change, and suddenly every pod's first lookup times out and every service call fails in a way that looks like the network is down. The other classic is ndots — a short name like api makes the resolver try api.namespace.svc.cluster.local, then svc.cluster.local, then cluster.local, then the search domains, so one lookup becomes five queries and a slow DNS server turns into slow everything. My line: DNS is the cluster's phone book, CoreDNS is the operator, and a DNS outage looks exactly like a network outage until you check it first."

```bash
# The fastest DNS check in a cluster — run it from a pod, not from your laptop:
kubectl run -it --rm dns-test --image=busybox:1.36 --restart=Never -- nslookup api.payments.svc.cluster.local
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs deploy/coredns --tail=20  # SERVFAIL and timeouts show up here
```

**Key Point:** "Pods resolve through CoreDNS behind the kube-dns Service, reading Services from the API — when CoreDNS is down or slow, every service call fails like a network outage, so check DNS first."

---

### Q10: NetworkPolicies — what do they actually enforce, and what do people get wrong?

**How to Answer:**

"I'd start with the two facts that surprise people. One: by default every pod in Kubernetes can talk to every other pod — there is no firewall until you create a policy. Two: a NetworkPolicy only works if your CNI enforces it — Flannel doesn't, Calico and Cilium do — so the same YAML can be a real firewall or a decoration depending on your CNI. A policy selects pods by label and then allows traffic from or to specific sources; everything not allowed is dropped, and the default is deny-all once any policy selects the pod. The mistake I call out is writing a policy that blocks the DNS port — pods need UDP 53 to CoreDNS, and blocking it breaks name resolution in a way that looks like a network failure. My line: default allow-all is the starting point, the CNI decides whether your policy is real, and always allow DNS before you lock anything down."

```yaml
# The pattern every team should start with: default-deny, then allow DNS, then allow the app.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-default-deny
  namespace: payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

**Key Point:** "By default every pod can reach every pod — a NetworkPolicy only works if your CNI enforces it, and the classic mistake is a deny-all that also blocks UDP 53 to CoreDNS, so always allow DNS first."

---

### Q11: "My pod can't reach the Service" — how do you debug it, step by step?

**How to Answer:**

"I'd work the chain in order and stop at the first broken link. One: can the pod resolve the name — nslookup from the pod, because a DNS failure looks like a network failure. Two: does the Service have endpoints — kubectl get endpoints, because a Service with no matching pods or a failing readiness probe has nothing to send traffic to. Three: can the pod reach the pod IP directly, bypassing the Service — if that works, kube-proxy or the CNI's service handling is the problem, not the app. Four: is a NetworkPolicy dropping it — check the policies selecting both pods and look for dropped packets. Five: is the app actually listening — curl the container port from inside the pod and check the container logs. The interview line: DNS, endpoints, pod-to-pod, policy, app — five checks, in that order, and you never skip DNS, because it's the one that lies to you."

```bash
# The five-check runbook, in order — run from a debug pod in the same namespace:
kubectl run -it --rm net-debug --image=nicolaka/netshoot --restart=Never -- bash
# inside the pod:
nslookup api.payments.svc.cluster.local   # 1. DNS
curl -v http://api.payments:8080/healthz  # 2-5. Service, then pod IP, then the app
# from your laptop, in parallel:
kubectl get endpoints api -n payments     # empty endpoints = no healthy pods behind the Service
kubectl get networkpolicy -n payments     # a policy selecting the pods can silently drop it
```

**Key Point:** "Debug in order — DNS, endpoints, pod-to-pod, NetworkPolicy, app — and never skip DNS, because a DNS failure looks exactly like a network failure."

---

### Q12: Walk me through designing networking for a multi-tenant platform on Kubernetes.

**How to Answer:**

"I'd take a platform where five teams share one cluster, because the design choices get obvious fast. One namespace per team, and a default-deny NetworkPolicy in each one so teams can't reach each other's pods unless a policy explicitly allows it. Pod CIDRs per node pool with Calico or Cilium so I get real policy enforcement and observability, and a CNI that supports eBPF so service load balancing stays fast at scale. One Gateway per environment owned by the platform team, with each team's HTTPRoute in their own namespace, so a bad route can't take down another team's front door. DNS gets enough CoreDNS replicas with autoscaling, and I set the pod dnsConfig ndots to 2 to stop the five-query lookup storm on short names. And every piece gets a runbook for the debug chain from Q11, because the network that nobody can debug is the network that stays broken at 3 AM. The interview close: namespaces for isolation, policy for the walls, Gateway API for the front door, and DNS sized like it matters — because it does."

**Key Point:** "Namespaces per team, default-deny policies, a CNI that enforces them, one platform-owned Gateway with per-team HTTPRoutes, and CoreDNS sized like it matters — because a network nobody can debug stays broken at 3 AM."
