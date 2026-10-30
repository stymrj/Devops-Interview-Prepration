# GitOps at Scale — Cheat Sheet

## The one-liners

| Concept | One line |
|---|---|
| GitOps at scale | Services × clusters × environments outgrows hand-written Applications |
| ApplicationSet | A factory: one template + generator = one Application per input |
| List generator | Hardcoded inputs, one Application each — small, explicit fleets |
| Git directories generator | Every folder under a path becomes an Application |
| Git files generator | JSON/YAML file contents become template parameters |
| Cluster generator | One Application per registered cluster matching a label selector |
| Matrix generator | Cross product of two generators — services × clusters in one file |
| Progressive syncs | Template changes roll out in batches, not fleet-wide |
| Generated-app count | The fastest health signal: wrong count = generator bug |

## Generators at a glance

| Generator | Input source | Best for | Trap |
|---|---|---|---|
| List | Hardcoded elements | Small, fixed fleets; learning | Editing the set per service |
| Git (directories) | Folders in a repo | Service onboarding via `mkdir` | Repo layout becomes API |
| Git (files) | JSON/YAML files | Per-service config (replicas, versions) | Schema drift across files |
| Clusters | Registered clusters + labels | Multi-cluster fleets | Mislabeled cluster gets wrong apps |
| Matrix | Two generators combined | Services × clusters explosion | Unfiltered cross product explodes |
| Pull Request | Open PRs | Ephemeral preview environments | Stale PRs leave orphan apps |
| SCM Provider | Org repos (GitHub/GitLab) | Whole-org onboarding | Needs provider credentials + pagination care |

## Template parameters you'll actually use

| Parameter | Comes from | Example use |
|---|---|---|
| `{{path.basename}}` | Git directories | App name = service folder name |
| `{{path}}` | Git directories | Chart path in the repo |
| `{{name}}`, `{{server}}` | Cluster generator | Destination cluster per app |
| `{{values.environment}}` | List / Git files | Values file per environment |
| `{{metadata.labels.x}}` | Cluster labels | Filtering which clusters get an app |

## Fleet design: 200 services × 5 clusters

| Decision | Choice |
|---|---|
| Repo layout | One directory per service, chart + values file per env |
| Generator | Matrix: Git directories × Cluster generator (label-filtered) |
| Projects | One ArgoCD Project per team, with RBAC |
| Rollouts | Progressive syncs: canary cluster, then batches |
| Secrets | External Secrets Operator — references only, never values in Git |
| Guardrail | CI check fails the PR if generated-app count jumps unexpectedly |
| Onboarding | Developer adds a directory and merges — app appears on its own |

## Debugging split

| Symptom | Likely cause | Check first |
|---|---|---|
| Zero / wrong number of apps | Generator or selector wrong | ApplicationSet status + `argocd appset get` |
| Right count, apps failing to sync | Template wrong | One failing app's sync error |
| Apps exist but stale | Generation not reconciling | ApplicationSet controller logs |
| Hand-edited app differs from Git | Drift on the Application itself | `argocd app diff` + sync policy |
| Sudden fleet-wide failure | Bad template change, no batching | Roll back template, add progressive syncs |

## Interview traps

| Trap | Correct answer |
|---|---|
| "Just write one Application per service" | Works to ~10 services; then repetition + drift kill it |
| "Put secrets in ApplicationSet parameters" | Parameters are for config — secrets leak the moment the app is stored |
| "Matrix without selectors" | Cross product explodes — filter combinations deliberately |
| "One big-bang template change" | Gate generation and sync separately; roll out in batches |
| "Copy the template per environment" | Differences belong in parameters/values files, never forked templates |
| "Debug from the failing app first" | Check the ApplicationSet's own status first — generation vs sync are separate failures |
