# Policy as Code (OPA & Kyverno) — Cheat Sheet

One page. Read it in the cab on the way to the interview.

## Core picture

| Piece | Job |
|-------|-----|
| Policy as code | Rules in Git, enforced by machine at admission — not a wiki page |
| RBAC | Who can do what — not whether the workload is safe |
| OPA | Rego decision engine; works on Kubernetes, Terraform, CI, APIs |
| Gatekeeper | OPA as a Kubernetes admission webhook |
| Kyverno | Kubernetes-native policies as plain YAML ClusterPolicies |

## OPA & Rego

- Feed OPA input + policy, get back allow or deny
- `deny[msg]` rules loop over input (e.g. `spec.containers[_]`) and return a message
- Test before deploying: `opa test policy.rego policy_test.rego -v`
- Gatekeeper split: **ConstraintTemplate** = reusable Rego logic, **Constraint** = that logic + real parameters

## Kyverno rule types

- **validate** = block bad things (latest tag, missing label, hostPath)
- **mutate** = fix defaults in place (add label, inject sidecar, set limits)
- **generate** = create companion objects (NetworkPolicy per new namespace)
- Match on kinds, validate with `pattern` or JMESPath `deny.conditions`
- `validationFailureAction: Audit` logs to PolicyReport; `Enforce` blocks

## Rollout order (the safe one)

1. Audit mode — collect violations, fix workloads
2. Enforce per-namespace, non-prod first
3. Then prod — never enforce a brand-new policy on day one

## Exceptions done right

- Kyverno: `exclude` block on a rule (namespace, workload) — visible, reviewable
- Gatekeeper: Constraint `exemptions` / namespaceSelector skips
- Never: delete the policy, or hand out cluster-admin to bypass it

## Debugging a denial

1. Read the error — it names the policy and rule
2. `kubectl get/describe clusterpolicy <name>` (Kyverno) or the Constraint (Gatekeeper)
3. Fix the workload, or fix the policy in Git and let it roll out
4. Never bypass the webhook by hand — the next deploy fails again

## OPA vs Kyverno in one line

- **Kyverno:** YAML-native, Kubernetes-only, fastest to adopt, has mutate/generate
- **OPA:** Rego, one engine for every system, steeper curve, wider reach

## Traps interviewers check

- RBAC ≠ policy: RBAC gates access, policy gates workload shape
- Vague denial messages create support tickets — messages must name the fix
- Audit-first rollout; enforcing day one blocks Friday deploys and kills trust
- Exceptions must be visible and time-boxed, not secret bypasses
