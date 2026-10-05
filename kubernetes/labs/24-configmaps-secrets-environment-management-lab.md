# ConfigMaps, Secrets & Environment Management — Hands-On Lab

Practical exercises for topic 24. You need a working cluster (minikube, kind, or any cloud cluster) and `kubectl` configured. All exercises are read-safe — no persistent changes outside your lab namespace.

Create the lab namespace first:

```bash
kubectl create namespace config-lab
kubectl config set-context --current --namespace=config-lab
```

---

## Exercise 1: Create a ConfigMap from literals and files

**Goal:** Build a ConfigMap two ways and inspect the difference.

**Commands:**

```bash
kubectl create configmap app-config --from-literal=log.level=debug --from-literal=app.port=8080
echo "port: 8080" > settings.yaml
kubectl create configmap file-config --from-file=settings.yaml
kubectl get configmap app-config -o yaml
```

**Expected output:** `app-config` shows `log.level` and `app.port` as key-values; `file-config` has one key `settings.yaml` with the file contents as its value.

**Why it matters:** Shows the two source patterns you'll use daily — literal flags for quick values, `--from-file` for real config files.

---

## Exercise 2: Inject ConfigMap keys as environment variables

**Goal:** Wire ConfigMap data into a Pod's environment.

**Commands:**

```bash
kubectl run env-demo --image=busybox --restart=Never --dry-run=client -o yaml \
  --env="LOG_LEVEL=$(kubectl get configmap app-config -o jsonpath='{.data.log\.level}')" > /dev/null
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'LOG|PORT'; sleep 30"]
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level
    - name: APP_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app.port
EOF
kubectl logs env-demo
```

**Expected output:** Logs print `LOG_LEVEL=debug` and `APP_PORT=8080`.

**Why it matters:** This is the canonical pattern for flat config — individual `configMapKeyRef` entries map keys to env vars.

---

## Exercise 3: Mount a ConfigMap as a volume

**Goal:** Consume ConfigMap data as files on disk.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls /etc/config && cat /etc/config/settings.yaml && sleep 60"]
    volumeMounts:
    - name: cfg
      mountPath: /etc/config
  volumes:
  - name: cfg
    configMap:
      name: file-config
EOF
kubectl logs volume-demo
```

**Expected output:** Lists `/etc/config/settings.yaml` and prints its contents.

**Why it matters:** Apps that expect real config files on disk (nginx.conf, app.properties) get them without image changes.

---

## Exercise 4: Watch a volume-mounted ConfigMap update live

**Goal:** Prove that volume mounts sync ConfigMap changes automatically.

**Commands:**

```bash
kubectl patch configmap file-config --type merge -p '{"data":{"settings.yaml":"port: 9090"}}'
sleep 75
kubectl exec volume-demo -- cat /etc/config/settings.yaml
```

**Expected output:** The file now shows `port: 9090` without any Pod restart — kubelet synced the update (takes ~60s).

**Why it matters:** Demonstrates the update path for volumes vs env vars (the classic interview trap from Q3).

---

## Exercise 5: Prove env vars do NOT update automatically

**Goal:** Confirm the env-var freeze behavior from Q3.

**Commands:**

```bash
kubectl patch configmap app-config --type merge -p '{"data":{"log.level":"error"}}'
kubectl exec env-demo -- printenv LOG_LEVEL   # still 'debug'
```

**Expected output:** `LOG_LEVEL` still prints `debug` — the running Pod is frozen at its start-time values.

**Why it matters:** You can demo this trap live in an interview. Fix it with a rollout: `kubectl rollout restart` on the Deployment.

---

## Exercise 6: Create and consume a Secret

**Goal:** Store a password in a Secret and inject it into a Pod.

**Commands:**

```bash
kubectl create secret generic db-creds --from-literal=password='s3cr3t-pass' --from-literal=username=dbadmin
kubectl get secret db-creds -o yaml        # note: base64, NOT encrypted
echo "czNjcjN0LXBhc3M=" | base64 -d        # decodes instantly
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo \"user=$DB_USER pass=$DB_PASS\" | sed 's/pass=.*/pass=<hidden>/'; sleep 60"]
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-creds
          key: password
EOF
kubectl logs secret-demo
```

**Expected output:** The Secret YAML shows base64 strings (decodable by anyone); the Pod logs show the injected values.

**Why it matters:** Makes the "base64 is encoding, not encryption" lesson visceral — you decode it yourself.

---

## Exercise 7: Verify Secrets land on tmpfs when volume-mounted

**Goal:** Confirm the tmpfs security property of Secret volumes.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "mount | grep /etc/secrets && cat /etc/secrets/password && sleep 60"]
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: db-creds
EOF
kubectl logs secret-volume-demo
```

**Expected output:** `mount` shows `/etc/secrets` is type `tmpfs` — the password never touches the node's disk.

**Why it matters:** This is the concrete security difference vs ConfigMaps — Secrets stay in RAM on the node.

---

## Exercise 8: Use an immutable ConfigMap

**Goal:** Create an immutable ConfigMap and try (and fail) to update it.

**Commands:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: frozen-config
immutable: true
data:
  mode: "production"
EOF
kubectl patch configmap frozen-config --type merge -p '{"data":{"mode":"staging"}}'
```

**Expected output:** The patch fails with an error like `spec.immutable: Invalid value: ... cannot update` — you must delete and recreate instead.

**Why it matters:** Shows the safety property from Q10: no silent config drift, every change is a new versioned object.

---

## Exercise 9: Simulate the committed-secret incident response

**Goal:** Practice the Q13 playbook — rotate, don't just delete.

**Commands:**

```bash
# Simulate: the password "leaked". Rotate it:
kubectl create secret generic db-creds --from-literal=password='n3w-r0tated-pass' --from-literal=username=dbadmin --dry-run=client -o yaml | kubectl apply -f -
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d; echo
```

**Expected output:** The new base64 value decodes to the rotated password; old value is gone.

**Why it matters:** Rotation is the real fix for a leaked secret — deleting git history alone helps nobody.

---

## Exercise 10: Cleanup

**Goal:** Tear down the lab cleanly.

**Commands:**

```bash
kubectl delete namespace config-lab
kubectl config set-context --current --namespace=default
```

**Expected output:** The namespace and everything in it is deleted; your context is back to default.

**Why it matters:** ConfigMaps and Secrets are namespace-scoped — deleting the namespace is the fastest full cleanup.
