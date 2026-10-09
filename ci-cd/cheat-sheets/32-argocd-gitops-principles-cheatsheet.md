# ArgoCD & GitOps — Cheat Sheet

## Install & access

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
argocd login localhost:8080 --username admin --insecure
```

## Application manifest (the core resource)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: main          # branch, tag, or commit SHA
    path: apps/payments/overlays/prod
    helm:                        # or kustomize: / directory:
      valueFiles: [values-prod.yaml]
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:                   # auto-apply Git changes
      prune: true                # delete resources removed from Git
      selfHeal: true             # revert manual kubectl edits
  ignoreDifferences:             # silence false drift
    - group: apps
      kind: Deployment
      jsonPointers: [/spec/replicas]   # HPA-owned field
```

## argocd CLI essentials

```bash
argocd app create myapp --repo <git-url> --path <dir> --dest-server https://kubernetes.default.svc --dest-namespace ns
argocd app get myapp                     # sync + health status
argocd app diff myapp                    # field-level Git-vs-live diff
argocd app sync myapp                    # manual sync
argocd app sync myapp --prune            # sync + delete orphaned resources
argocd app set myapp --self-heal         # enable drift auto-correction
argocd app get myapp --hard-refresh      # bypass repo-server cache
argocd app history myapp                 # sync history (what commit went live when)
argocd app rollback myapp <id>           # roll back to a sync revision
argocd cluster add <context>             # register a cluster
argocd repo add <git-url>                # register a private repo
```

## Sync waves (apply ordering)

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # lower = applied first
    # "-2" CRDs, "-1" namespaces/config, "0" default, "1" jobs that run last
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
```

## Sync options (annotations)

| Annotation | Effect |
|---|---|
| `Prune=false` | Keep resource even when removed from Git |
| `Replace=true` | `kubectl replace` instead of apply (immutable fields) |
| `Force=true` | Delete + recreate on apply failure |
| `ServerSideApply=true` | Use server-side apply for the resource |

## ApplicationSet (fleet templating)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata: {name: fleet, namespace: argocd}
spec:
  generators:
  - list:                                  # or: clusters, git (dirs/files), matrix, merge
      elements:
      - {env: dev, ns: team-a-dev}
      - {env: prod, ns: team-a-prod}
  template:
    metadata: {name: 'team-a-{{env}}'}
    spec:
      source: {repoURL: <git>, path: 'apps/team-a/{{env}}'}
      destination: {server: https://kubernetes.default.svc, namespace: '{{ns}}'}
```

## Projects (RBAC scoping)

```bash
argocd proj create team-a -d https://kubernetes.default.svc,team-a-\* -s https://github.com/myorg/gitops-repo
argocd proj add-source team-a 'https://github.com/myorg/team-a-*'
argocd proj add-destination team-a 'https://kubernetes.default.svc' 'team-a-*'
argocd proj role create team-a deployer
argocd proj role add-policy team-a deployer -p 'allow' -r 'applications' -o 'sync' -n 'team-a-*'
```

## Status vocabulary

| Term | Meaning |
|---|---|
| Synced / OutOfSync | Git desired state vs live cluster match |
| Healthy / Degraded / Progressing | Runtime state of the resources |
| Prune | Deleting resources removed from Git |
| Self-heal | Reverting manual cluster edits back to Git state |
| Drift | Any live-vs-Git difference |
| Target revision | The branch/tag/SHA the app tracks |

## Quick troubleshooting

```bash
argocd app get myapp --show-operation    # why the last sync failed
kubectl -n argocd logs deploy/argocd-application-controller | grep myapp
kubectl -n argocd logs deploy/argocd-repo-server   # template/render errors
argocd admin settings resource-overrides health ... # custom health checks
```
