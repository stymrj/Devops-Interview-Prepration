# Kubernetes Networking Deep Dive — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| Pod network | Flat — every pod gets its own IP and reaches every other pod directly, no NAT |
| CNI plugin | The contract the kubelet calls at pod create/delete to wire up networking |
| Flannel | Simple VXLAN overlay — easy to run, no policy, no observability |
| Calico | BGP routing + NetworkPolicy enforcement — the solid production default |
| Cilium | eBPF in the kernel — service LB, policy, observability, hardest to debug |
| kube-proxy | Writes iptables/IPVS rules on every node so a ClusterIP rewrites to a pod IP |
| ClusterIP | Inside the cluster only — the default, what internal services use |
| NodePort | Opens a port on every node — a debugging hack, never for real users |
| LoadBalancer | Provisions a real cloud load balancer — the production path for public traffic |
| Ingress | The routing rules (host, path, backend) — a wish list without a controller |
| Ingress controller | The real router (NGINX, Traefik) that reads Ingress rules and forwards traffic |
| Gateway API | Layered successor — Gateway (front door, platform-owned) + HTTPRoute (app-owned) |
| CoreDNS | The cluster's phone book — resolves Service names from API objects |
| ndots | Short names try five suffixes — one lookup becomes five queries on a slow DNS |
| NetworkPolicy | A firewall by label — only real if your CNI enforces it |
| Default-deny | Once a policy selects a pod, everything not allowed is dropped |
| DNS port | UDP/TCP 53 to CoreDNS — the rule every deny-all policy must allow first |

## CNI comparison

| CNI | How it moves packets | Policy | Best for |
|---|---|---|---|
| Flannel | VXLAN overlay | No | Learning, small clusters |
| Calico | BGP routes (or VXLAN) | Yes | Solid production default |
| Cilium | eBPF in the kernel | Yes | Speed, observability, scale |
| kindnet | Simple bridge | No | Local kind clusters only |

Flannel is a simple overlay, Calico adds BGP and policy, Cilium does it in eBPF — and a NetworkPolicy is a decoration until the CNI enforces it.

## Service types

| Type | Reachable from | Use it for |
|---|---|---|
| ClusterIP | Pods in the cluster | Databases, internal APIs |
| NodePort | Any node IP + port (30000–32767) | Debugging only |
| LoadBalancer | A real external IP | Anything public |
| Headless (ClusterIP: None) | Direct pod IPs via DNS | StatefulSets, direct pod discovery |

ClusterIP inside, LoadBalancer outside, NodePort only for debugging.

## kube-proxy modes

| Mode | How it picks a backend | Trade-off |
|---|---|---|
| iptables | Walks a rule chain, one by one | Simple, slow at scale |
| IPVS | Hashes to the right backend | Fast at 1000+ services |
| eBPF (Cilium) | In-kernel, no kube-proxy | Fastest, needs Cilium |

A Service is a stable name over rules on every node — the mode decides whether it stays fast at a thousand services.

## Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---|---|
| Shape | One flat resource, all the config | Layered: Gateway + HTTPRoute + more |
| Extensions | Vendor annotations (differ per controller) | First-class fields |
| Ownership | One resource, teams collide | Platform owns Gateway, apps own routes |
| Traffic splitting | Annotation hack | Built-in weights |
| Header matching | Annotation hack | Built-in |
| Cross-namespace | Limited | First-class references |

Ingress is one flat rulebook with vendor annotations; Gateway API splits the front door from the routes.

## TLS at the ingress

| Piece | What it does | Classic mistake |
|---|---|---|
| Secret | Holds the cert + key | Secret in the wrong namespace — Ingress can't read it |
| cert-manager | Issues + renews from Let's Encrypt | No alerting when issuance fails |
| Termination | TLS ends at the controller | Plain HTTP behind the ingress for sensitive traffic |
| Wildcard cert | One cert for many hosts | Shared across teams, no rotation plan |

Automate the certs with cert-manager, keep the Secret next to the Ingress, and encrypt the hop behind it too if the traffic is sensitive.

## Canary at the ingress

| Method | How it splits | Rollback |
|---|---|---|
| NGINX Ingress canary | Second Ingress + weight annotation | Set weight to 0 |
| Gateway API | One HTTPRoute, two weighted backends | Set weight to 0 |
| What to watch | Error rate + latency on the canary | Flip in seconds, no deploy |

Split at the ingress, watch the canary, roll back with a weight change — but the canary still writes to the same database, so a bad migration hurts everyone.

## DNS in the cluster

| Name | Resolves to |
|---|---|
| `api` | `api.<namespace>.svc.cluster.local` (search domains expand it) |
| `api.payments` | `api.payments.svc.cluster.local` |
| `api.payments.svc.cluster.local` | The Service's ClusterIP |
| Headless service | The pod IPs, one A record each |

DNS is the cluster's phone book and CoreDNS is the operator — a DNS outage looks exactly like a network outage until you check it first.

## NetworkPolicy essentials

| Rule | Why |
|---|---|
| Default is allow-all | No policy means any pod can reach any pod |
| Policy only works if the CNI enforces it | Flannel ignores it; Calico and Cilium enforce it |
| Default-deny first | One policy selecting a pod denies everything not allowed |
| Always allow DNS | UDP/TCP 53 to kube-dns, or name resolution breaks |
| Select by label | Policies attach to pods by label, not by name |

The classic mistake is a deny-all that also blocks UDP 53 to CoreDNS.

## The five-check debug chain

| # | Check | Command | What a failure means |
|---|---|---|---|
| 1 | DNS | `nslookup api.payments.svc.cluster.local` (from a pod) | CoreDNS down, or the name is wrong |
| 2 | Endpoints | `kubectl get endpoints api -n payments` | No healthy pods behind the Service |
| 3 | Pod-to-pod | `curl` the pod IP directly | kube-proxy / CNI service handling |
| 4 | Policy | `kubectl get networkpolicy -n payments` | A policy is silently dropping it |
| 5 | App | `curl` the container port + container logs | The app isn't listening |

DNS, endpoints, pod-to-pod, policy, app — in that order, and never skip DNS, because it's the one that lies to you.

## Multi-tenant networking design

| Decision | Choice | Why |
|---|---|---|
| Isolation | One namespace per team | Boundaries you can enforce |
| Walls | Default-deny NetworkPolicy per namespace | Teams can't reach each other's pods by accident |
| CNI | Calico or Cilium | Policy enforcement + observability |
| Front door | One Gateway per environment, platform-owned | A bad route can't take down another team |
| Routes | HTTPRoute per team, in their namespace | Teams own their own routing |
| DNS | Enough CoreDNS replicas + autoscaling, ndots: 2 | DNS sized like it matters — because it does |

Namespaces for isolation, policy for the walls, Gateway API for the front door, and DNS sized like it matters.
