# Kubernetes NetworkPolicies Cheat Sheet

Dense reference for policy syntax, selectors, and the debugging commands that
pair with the [full guide](../52-kubernetes-networkpolicies.md).

## Policy Skeleton

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <name>
  namespace: <ns>          # selectors are namespace-scoped
spec:
  podSelector: {}           # which pods this policy applies to ({} = all in ns)
  policyTypes: [Ingress, Egress]
  ingress: []               # allowed inbound rules
  egress: []                # allowed outbound rules
```

## Selector Types

| Selector | Matches | Typical use |
|---|---|---|
| `podSelector` | Pods in the same namespace | App tiers (frontend → api) |
| `namespaceSelector` | Pods in matching namespaces | Monitoring scraping across namespaces |
| `ipBlock: { cidr: 10.0.0.0/8 }` | Raw IPs/CIDRs outside cluster | Office VPN, partner endpoints |
| `ipBlock: { cidr: ..., except: [...] }` | CIDR minus exclusions | Allow a subnet, block one host in it |

- `namespaceSelector` matches labels on the **namespace object** — label namespaces at creation (`kubectl label ns prod team=platform`).
- Combining `namespaceSelector` + `podSelector` in one `from`/`to` entry = AND. Multiple entries in the list = OR.
- Policies are **additive (union)** — no deny rules, no ordering.

## Ready-Made Policies

Default deny-all (ingress + egress):

```yaml
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

Allow frontend → api on 8080:

```yaml
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Ingress]
  ingress:
    - from: [{ podSelector: { matchLabels: { app: frontend } } }]
      ports: [{ port: 8080 }]
```

Allow Prometheus (other namespace) to scrape port 9090:

```yaml
spec:
  podSelector: { matchLabels: { app: myapp } }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { team: monitoring } }
          podSelector: { matchLabels: { app: prometheus } }
      ports: [{ port: 9090 }]
```

Egress with DNS exception (the one you always forget):

```yaml
spec:
  podSelector: { matchLabels: { app: api } }
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { kubernetes.io/metadata.name: kube-system } }
          podSelector: { matchLabels: { k8s-app: kube-dns } }
      ports:
        - { port: 53, protocol: UDP }
        - { port: 53, protocol: TCP }
    - to: [{ podSelector: { matchLabels: { app: db } } }]
      ports: [{ port: 5432 }]
```

Allow external HTTPS to one CIDR:

```yaml
egress:
  - to: [{ ipBlock: { cidr: 203.0.113.0/24 } }]
    ports: [{ port: 443, protocol: TCP }]
```

## kubectl Commands

| Task | Command |
|---|---|
| List policies | `kubectl get networkpolicy -A` |
| Describe (shows matched pods + rules) | `kubectl describe networkpolicy <name> -n <ns>` |
| Dump as YAML | `kubectl get networkpolicy <name> -n <ns> -o yaml` |
| Delete | `kubectl delete networkpolicy <name> -n <ns>` |
| Label a namespace (for namespaceSelector) | `kubectl label ns prod team=platform` |
| Check CNI in use | `kubectl get pods -n kube-system` (calico-node / cilium = enforced) |
| Test connectivity pod-to-pod | `kubectl run tmp --image=busybox -n <ns> -- wget -q -T 5 -O /dev/null <url>` |

## Interview One-Liners

- Default K8s networking is flat — every pod reaches every pod until a policy says otherwise.
- Empty `podSelector: {}` = the whole namespace. That's your default-deny.
- Policies stack by union; there is no deny and no rule order.
- Flannel doesn't enforce NetworkPolicies. Calico and Cilium do.
- Egress isolation breaks DNS — always carve out port 53 to CoreDNS.
- Debug order: label match → CNI enforcement → direction (ingress vs egress).
- L3/L4 only: for identity-based or L7 rules, bring a service mesh.
