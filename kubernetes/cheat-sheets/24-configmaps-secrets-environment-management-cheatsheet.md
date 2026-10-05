# ConfigMaps, Secrets & Environment Management — Cheat Sheet

Topic 24 of 58. Dense reference for interviews and on-call.

## Create ConfigMaps

```bash
kubectl create configmap app-config --from-literal=LOG_LEVEL=debug --from-literal=PORT=8080
kubectl create configmap app-config --from-file=settings.yaml        # whole file as one key
kubectl create configmap app-config --from-file=config/              # every file in dir = one key
kubectl create configmap app-config --from-env-file=.env            # KEY=VALUE lines
```

## Inspect ConfigMaps

```bash
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o jsonpath='{.data}'              # raw key-values
```

## Env-var injection

```yaml
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef: { name: app-config, key: LOG_LEVEL }   # single key
envFrom:
  - configMapRef: { name: app-config }                         # ALL keys at once
  # optional: true   → Pod starts even if the ConfigMap is missing
```

## Volume injection

```yaml
volumes:
  - name: cfg
    configMap:
      name: app-config
      items:                                    # optional: remap keys → filenames
        - key: settings.yaml
          path: app-settings.yaml
```

## ConfigMap updates

- Volume mounts: kubelet syncs changes in ~60s, **no restart needed**
- Env vars: frozen at container start, **must restart Pods** (`kubectl rollout restart deploy/web`)
- 1 MiB size limit per ConfigMap; stored plaintext in etcd
- `immutable: true` → cannot be updated, only recreated (forces versioned config)

## Create Secrets

```bash
kubectl create secret generic db-creds --from-literal=password='s3cr3t' --from-literal=user=admin
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key     # TLS type
kubectl create secret docker-registry reg-creds \                  # image pull
  --docker-server=registry.example.com --docker-username=u --docker-password=p
```

## Inspect Secrets (values are base64, NOT encrypted)

```bash
kubectl get secrets
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
echo "czNjcjN0" | base64 -d        # anyone can do this
```

## Secret injection (same mechanics as ConfigMaps)

```yaml
env:
  - name: DB_PASS
    valueFrom:
      secretKeyRef: { name: db-creds, key: password }
volumes:
  - name: secrets
    secret:
      secretName: db-creds          # mounted on tmpfs, never hits node disk
```

## Secret hardening checklist

- Secrets never in git → pre-commit hooks (gitleaks), push protection
- Encrypt etcd at rest (EncryptionConfiguration + KMS)
- Tight RBAC: who can `get`/`list` secrets per namespace
- Rotate on leak: `kubectl create secret ... --dry-run=client -o yaml | kubectl apply -f -`
- Prefer External Secrets Operator / Vault for rotation + audit

## Per-environment config

```bash
# Kustomize overlays: base manifest + env-specific overlays
kubectl apply -k overlays/production/
```

One image for all envs; common config in base, overrides in overlays.

## Debug missing env vars

```bash
kubectl describe pod <pod>            # shows resolved env at startup
kubectl get configmap <name> -o yaml  # does the key actually exist?
kubectl rollout restart deploy/<name> # env vars freeze at start — restart picks up changes
```
