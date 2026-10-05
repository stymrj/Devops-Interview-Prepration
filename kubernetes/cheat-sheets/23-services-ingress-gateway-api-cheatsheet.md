# Services, Ingress & Gateway API — Cheat Sheet

Topic 23 of 58. Dense reference for interviews and on-call.

## Create and manage Services

```bash
kubectl expose deployment web --port=80                    # ClusterIP Service
kubectl expose deployment web --port=80 --type=NodePort    # NodePort Service
kubectl expose deployment web --port=80 --type=LoadBalancer
kubectl create service clusterip api --tcp=80:8080         # imperative
kubectl delete svc web
```

## Inspect Services and Endpoints

```bash
kubectl get svc                         # all Services, types, ClusterIPs
kubectl get svc web -o wide             # node port, external IP
kubectl get endpoints web               # WHICH PODS get traffic — check first
kubectl get endpointslices              # newer EndpointSlice API
kubectl describe svc web                # selector + ports + events
kubectl get svc web -o jsonpath='{.spec.selector}'
```

## DNS (CoreDNS)

```bash
# from inside any Pod:
nslookup web                            # short name, same namespace
nslookup web.default.svc.cluster.local  # FQDN: <svc>.<ns>.svc.cluster.local
nslookup db-0.db-headless                # headless → individual Pod IPs
```

## Headless Service (clusterIP: None)

```yaml
spec:
  clusterIP: None        # DNS returns Pod IPs, no load balancing
  selector:
    app: postgres
```
Use for: StatefulSets, databases, Kafka — anything where each Pod is distinct.

## Service fields that bite

```yaml
spec:
  selector: {app: web}            # must match Pod labels or Endpoints = <none>
  ports:
  - port: 80                      # Service port (what clients hit)
    targetPort: 8080              # container port (what the app listens on)
    nodePort: 31234               # NodePort range 30000-32767
  sessionAffinity: ClientIP       # sticky sessions (usually leave as None)
  externalTrafficPolicy: Local    # preserves client IP; Cluster = SNAT
  type: ClusterIP                 # ClusterIP | NodePort | LoadBalancer | ExternalName
```

## Ingress essentials

```bash
kubectl get ingress                 # ADDRESS column = controller's entry point
kubectl describe ingress web        # resolved rules, backends, TLS
kubectl get ingressclass             # available controller classes
```

```yaml
spec:
  ingressClassName: nginx          # REQUIRED — picks the controller
  tls:
  - hosts: [app.example.com]
    secretName: app-tls            # MUST be in same namespace as Ingress
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix           # Prefix | Exact | ImplementationSpecific
        backend:
          service: {name: api-svc, port: {number: 80}}
```

## Ingress debugging checklist

```bash
kubectl get ingressclass                            # controller installed?
kubectl get pods -n ingress-nginx                   # controller running?
kubectl get endpoints api-svc                       # backend has Pods?
curl -H "Host: app.example.com" http://<ingress-ip>/api   # bypass DNS
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50
```

## Gateway API quick map

```bash
kubectl get gatewayclass       # which implementations exist
kubectl get gateway -A         # listener config (infra-owned)
kubectl get httproute -A       # app routing (team-owned)
```

| Resource | Owned by | Does |
|---|---|---|
| GatewayClass | platform | picks the controller implementation |
| Gateway | infra team | ports, hostnames, TLS listeners |
| HTTPRoute | app team | path/header match → backend Service |
| TCPRoute / TLSRoute | app team | non-HTTP traffic |

## kube-proxy modes

```bash
kubectl logs -n kube-system -l k8s-app=kube-proxy | head   # check mode
```
- **iptables** (default): DNAT rules per Service, fine at moderate scale
- **IPVS**: better performance with thousands of Services
- **nftables / eBPF (Cilium)**: modern, fewer moving parts

## Interview one-liners

- Empty Endpoints = selector mismatch. Always check `kubectl get endpoints` first.
- Ingress without a controller = YAML that does nothing.
- kube-proxy balances **connections**, not requests — long-lived connections pin to one Pod.
- `externalTrafficPolicy: Cluster` SNATs client IPs; `Local` preserves them.
- Ingress TLS Secret must live in the **same namespace** as the Ingress.
- Gateway API = structured CRDs + role split; Ingress = frozen but stable.
