# Helm Charts — Cheat Sheet

*Dense command reference. Keep it open in the next tab.*

## Vocabulary

| Term | Meaning |
|---|---|
| Chart | The package — templates + values + metadata |
| Release | A deployed instance of a chart (name + namespace + revision) |
| Repository | Where charts are stored (index.yaml or OCI registry) |
| Values | Inputs merged into templates at render time |
| Revision | Each install/upgrade/rollback creates one |

## Daily commands

```bash
helm create mychart                      # scaffold a new chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo postgres                # find charts
helm show values bitnami/postgres        # read defaults before installing

helm install myapp ./mychart             # install from local dir
helm install myapp bitnami/postgres -f prod.yaml
helm install myapp ./mychart --dry-run --debug   # render only

helm list                                # releases in current ns
helm list -A
helm status myapp
helm history myapp                       # all revisions

helm upgrade myapp ./mychart -f prod.yaml --wait --timeout 5m --atomic
helm rollback myapp 2                    # revert to revision 2
helm uninstall myapp
helm uninstall myapp --keep-history      # remove resources, keep record

helm template myapp ./mychart --set replicaCount=3   # render locally
helm lint ./mychart
helm package ./mychart                   # build .tgz
helm push mychart-0.1.0.tgz oci://registry.example.com/charts
```

## Values precedence (later wins)

```
values.yaml  <  -f file1  <  -f file2  <  --set  <  --set-string
```

`--set a.b.c=x` for nested, `--set-string` to keep `true123` a string.

## Template functions you'll actually use

```
{{ .Values.replicaCount }}               # value lookup
{{ .Values.image.tag | default "latest" }}  # fallback
{{ .Values.secret | quote }}             # "true123" not true123
{{ include "mychart.fullname" . | nindent 4 }}  # helper + indentation
{{ .Values.labels | toYaml | nindent 4 }}  # dump a map as YAML
{{- if .Values.ingress.enabled }}        # conditional resource
{{ range .Values.hosts }} ... {{ end }}  # loops
{{ required "set image.tag!" .Values.image.tag }}  # fail fast
```

Files in templates: `_helpers.tpl` (shared funcs), `NOTES.txt` (post-install text).

## Chart.yaml essentials

```yaml
apiVersion: v2
name: mychart
version: 0.2.0        # chart version — bump on EVERY chart change
appVersion: "1.24.0"  # app version — default image tag
dependencies:
  - name: postgresql
    version: "12.x.x" # pin majors, or suffer later
    repository: "oci://registry-1.docker.io/bitnamicharts"
```

`helm dependency update` → vendors into `charts/`.

## Hooks (fire once, not managed)

```yaml
annotations:
  "helm.sh/hook": pre-install,pre-upgrade
  "helm.sh/hook-weight": "-5"          # lower runs first
  "helm.sh/hook-delete-policy": hook-succeeded
```

Common: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`. Use for migrations; never for things you want tracked.

## Secrets — the rules

- Never plain secrets in values.yaml in git
- `--set password=x` leaks into shell history and CI logs
- Patterns: sops + helm-secrets (encrypted values), or External Secrets Operator (chart creates ExternalSecret CR, operator syncs from AWS Secrets Manager)

## Stuck-release rescue

```bash
helm history myapp                      # find a good revision
helm rollback myapp <good-revision>     # clears PENDING_UPGRADE
# nuclear option:
helm uninstall myapp
```

Prevention: `--atomic` (auto-rollback on failure), `--wait --timeout` (block until ready).

## Debug order

```
helm status myapp → helm history → helm template (render bug?)
→ helm lint → kubectl get events → kubectl describe/logs
```

## One-liners for the interview

- Helm 3 has **no Tiller** — it uses your kubeconfig directly, RBAC applies to you.
- Upgrades do a **3-way merge**: old chart + new chart + live state; hand-edits on unmanaged fields survive.
- Releases stored as **Secrets** (`sh.helm.release.v1.<name>.v<n>`) in the release namespace.
- Rollback re-applies a **stored manifest** — fast, but no backward data migrations.
- Chart `version` ≠ `appVersion`: one tracks the package, one tracks the app.
