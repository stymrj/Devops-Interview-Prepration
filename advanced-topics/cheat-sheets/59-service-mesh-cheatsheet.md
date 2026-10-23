# Service Mesh Deep Dive — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## Core picture

| Piece | Job |
|-------|-----|
| Data plane | Envoy sidecars — move traffic, enforce policy per request |
| Control plane | istiod (Istio) — pushes config + certificates to sidecars |
| Injection | Label namespace -> webhook adds `istio-proxy` container |
| North-south | Gateway / ingress — users entering the cluster |
| East-west | Mesh — services talking inside the cluster |

## Istio objects (say what each one decides)

- **VirtualService** — where traffic goes: weights, mirrors, faults, retries, timeouts
- **DestinationRule** — how it goes there: subsets (v1/v2), load balancing, circuit breaking
- **Gateway** — edge entry: hosts, ports, TLS termination
- **PeerAuthentication** — mTLS mode: DISABLE / PERMISSIVE / STRICT
- **AuthorizationPolicy** — who may call whom, by service-account identity

## Traffic recipes

- Canary: VirtualService weights 95/5 -> watch canary error rate + p99 -> shift in steps
- Retries: idempotent only, 2-3 attempts, backoff, `retryOn: 5xx` — blind retries amplify outages
- Circuit breaking: `outlierDetection` ejects failing hosts; `connectionPool` caps blast radius
- Fault injection: `httpFault` delay/abort — the chaos-testing hook built into the mesh

## mTLS rollout

`PERMISSIVE (accept plain + TLS) -> prove all traffic encrypted -> STRICT (TLS only)` — never flip STRICT on day one.

## Istio vs Linkerd

- Linkerd: lighter, K8s-only, fast install, less CPU — pick for simple
- Istio: richer policy, multi-cluster, VMs — pick for power, pay in ops weight

## Debug checklist

```bash
kubectl get pod <pod>                 # 2/2 Ready? sidecar present?
istioctl analyze -n <ns>              # config errors first
istioctl proxy-config routes <pod>    # what routes does Envoy actually have?
kubectl logs <pod> -c istio-proxy     # access logs, both caller and receiver
istioctl x describe pod <pod>         # mTLS mode + policy applied
```

Classic bug: mTLS mode mismatch (one side STRICT, other plain) -> 503s with working app code.

## Lines that land

- "Gateway handles north-south, mesh handles east-west — they complement, not replace."
- "Control plane down is not data plane down — traffic flows, config just stops updating."
- "VirtualService decides where, DestinationRule decides how."
- "A mesh is worth it at many services and shared policy — at three services it's overkill, and I say the cost out loud."

## Red flags to avoid

- Recommending a mesh without naming its CPU/latency/ops cost
- Retries on non-idempotent calls
- Flipping STRICT mTLS before a PERMISSIVE migration
- Debugging the app first when the sidecar config is the suspect
