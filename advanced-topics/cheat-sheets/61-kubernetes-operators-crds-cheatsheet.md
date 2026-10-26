# Kubernetes Operators & CRDs — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## Core picture

| Piece | Job |
|-------|-----|
| CRD | Teaches the API server a new resource type: group, kind, versioned schema |
| Custom resource | An instance of that type — the user's declaration of intent |
| Controller | Loop that watches the resource and drives reality toward spec |
| Reconciliation | Watch -> compare desired vs actual -> act -> requeue, forever |
| Operator | Controller + domain knowledge for one app (backups, failover, upgrades) |

## CRD anatomy

- `group` (e.g. `db.example.com`), `kind`, `plural`, `scope` (Namespaced vs Cluster)
- `versions[]`: each with `served`, `storage`, and an `openAPIV3Schema`
- One storage version only — etcd stores that shape, others convert
- Schema validation runs at the API server, before your code sees the object

## Spec vs status

- **spec** = desired state, written by the user, never by the controller
- **status** = observed state, written by the controller via `Status().Update()`
- Conditions + phase in status are what `kubectl describe` users actually read

## Controller rules

- **Level-based, not edge-based:** act on current state, not on remembered events
- Requeue on every pass; never assume your last action worked
- `SetControllerReference` = ownership link -> garbage collection + watch events
- Watch child resources (Deployments, Services) so drift triggers a reconcile

## Finalizers

- Added on first reconcile; block deletion while cleanup runs
- Use only for external cleanup (cloud DB, bucket, DNS record)
- Stuck Terminating = finalizer never removed — fix the controller, not the object

## Building

- **Kubebuilder** = default scaffold (Go, controller-runtime, CRD + RBAC generation)
- **Operator SDK** = same runtime + Helm/Ansible-based operator options
- `make manifests` regenerates CRDs/RBAC; `make install` applies them
- RBAC markers: `//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=...`

## Versioning

- v1alpha1 -> v1beta1 -> v1; additive changes are free
- Renames/removals need a conversion webhook + migration window
- Never serve two versions without deciding which one is storage

## Debugging checklist

1. `kubectl describe <cr> <name>` — conditions, events, status
2. Controller logs — reconcile errors, requeue backoff
3. RBAC — `forbidden` errors are the most common silent failure
4. Finalizers — stuck Terminating means cleanup never finished
5. CRD installed? `kubectl get crds | grep <group>`

## When NOT to build one

- Stateless app with no day-2 steps -> Deployment + Helm is enough
- One-off automation -> a Job or CronJob, not a whole API extension
- The runbook fits on one page with no ordering constraints -> skip it
