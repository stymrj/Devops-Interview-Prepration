# Hands-On Lab: Persistent Storage in Kubernetes

Practice persistent volumes, claims, StorageClasses, and StatefulSets. You need a cluster — `kind`, `minikube`, or any managed cluster works. Check it first:

```bash
kubectl cluster-info
kubectl get storageclass
```

---

## Exercise 1 — Prove container data is ephemeral

**Goal:** See data vanish when a container restarts.

```bash
kubectl run tmp --image=nginx --restart=Never
kubectl exec tmp -- sh -c 'echo "hello" > /usr/share/nginx/html/lost.txt'
kubectl exec tmp -- cat /usr/share/nginx/html/lost.txt
kubectl delete pod tmp
kubectl run tmp2 --image=nginx --restart=Never
kubectl exec tmp2 -- cat /usr/share/nginx/html/lost.txt || echo "GONE"
```

**Expected output:** `GONE` — the second Pod has none of the first Pod's files.

**Why it matters:** This is the foundational reason persistent storage exists. Interviewers expect you to have seen this, not just read about it.

---

## Exercise 2 — Mount an emptyDir and watch it survive a container restart

**Goal:** Understand pod-scoped temporary storage.

```bash
cat <<'EOF' > emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo cached > /cache/data.txt && sleep 3600"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}
EOF
kubectl apply -f emptydir-pod.yaml
kubectl exec cache-demo -- cat /cache/data.txt
```

**Expected output:** `cached` — the file is there even though the container process just writes once.

**Why it matters:** emptyDir is the answer for scratch space, caches, and sharing files between containers in a Pod. Know where it survives and where it doesn't.

---

## Exercise 3 — Create a PVC with dynamic provisioning

**Goal:** Get a real persistent volume bound to a claim.

```bash
cat <<'EOF' > data-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
EOF
kubectl apply -f data-pvc.yaml
kubectl get pvc lab-data -w
```

**Expected output:** STATUS flips from `Pending` to `Bound`, with a `VOLUME` name like `pvc-...`.

**Why it matters:** This is the everyday storage workflow. Note: on `kind`/`minikube` the default StorageClass provides hostPath-backed volumes — fine for learning.

---

## Exercise 4 — Mount the PVC into a Pod and write persistent data

**Goal:** Prove data survives a Pod deletion.

```bash
cat <<'EOF' > writer-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo persistent > /data/note.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: lab-data
EOF
kubectl apply -f writer-pod.yaml
kubectl exec writer -- cat /data/note.txt
kubectl delete pod writer
kubectl apply -f writer-pod.yaml
kubectl exec writer -- cat /data/note.txt
```

**Expected output:** `persistent` printed both times — the file survives the Pod being deleted and recreated.

**Why it matters:** This is the exact demo an interviewer respects: state outliving the compute. Contrast it mentally with Exercise 1.

---

## Exercise 5 — Expand the volume in place

**Goal:** Grow a PVC without recreating anything.

```bash
kubectl get storageclass -o jsonpath='{.items[0].metadata.name}'
kubectl patch pvc lab-data --type=merge -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl get pvc lab-data
```

**Expected output:** `CAPACITY` eventually shows `2Gi`. (Needs `allowVolumeExpansion: true` on the StorageClass; `kind`'s standard class may not support it — check `kubectl get sc standard -o yaml`.)

**Why it matters:** In-place expansion is the production answer to "the disk is full." Know the prerequisites: the flag, grow-only, and that you can't shrink.

---

## Exercise 6 — Deploy a StatefulSet with volumeClaimTemplates

**Goal:** Give each replica its own sticky disk.

```bash
cat <<'EOF' > web-stateful.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
EOF
kubectl apply -f web-stateful.yaml
kubectl get pvc
kubectl get pods -l app=web
```

**Expected output:** Two PVCs (`www-web-0`, `www-web-1`) and two Pods, each bound to its own.

**Why it matters:** `volumeClaimTemplates` is the signature StatefulSet feature. Each replica's data survives crashes and rescheduling.

---

## Exercise 7 — Prove StatefulSet storage survives rescheduling

**Goal:** Delete a StatefulSet Pod and watch it reclaim its data.

```bash
kubectl exec web-1 -- sh -c 'echo "replica-1-data" > /usr/share/nginx/html/id.txt'
kubectl delete pod web-1
kubectl exec web-1 -- cat /usr/share/nginx/html/id.txt
kubectl get pvc www-web-1
```

**Expected output:** `replica-1-data` — the new Pod (same name) re-attaches the same PVC.

**Why it matters:** Stable identity + sticky storage is why databases run on StatefulSets. This exercise is the whole argument in one demo.

---

## Exercise 8 — Take a volume snapshot and restore from it

**Goal:** Practice point-in-time backup and restore.

```bash
kubectl get volumesnapshotclass 2>/dev/null || echo "no snapshot classes installed"
```

If a snapshot class exists:

```bash
cat <<'EOF' > snap.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: lab-snap
spec:
  volumeSnapshotClassName: <your-class>
  source:
    persistentVolumeClaimName: lab-data
EOF
kubectl apply -f snap.yaml
kubectl get volumesnapshot lab-snap
```

**Expected output:** `READYTODISPLAY` becomes `true`.

**Why it matters:** Snapshots are the fast path for pre-migration backups and staging clones. Note this only works when the cluster has a snapshot controller and the CSI driver supports it — knowing that limitation is itself an interview answer.

---

## Exercise 9 — Debug a stuck PVC (the classic Pending)

**Goal:** Read the events and find the cause.

```bash
cat <<'EOF' > bad-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: doomed
spec:
  storageClassName: no-such-class
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 100Gi
EOF
kubectl apply -f bad-pvc.yaml
kubectl describe pvc doomed | grep -A5 Events
kubectl delete pvc doomed
```

**Expected output:** Events like `storageclass.storage.k8s.io "no-such-class" not found` — then clean it up.

**Why it matters:** PVC-stuck-in-Pending is one of the most common real-world storage tickets. The fix is always in the events; this builds the muscle memory of looking there first.

---

## Cleanup

```bash
kubectl delete statefulset web --cascade=orphan
kubectl delete pvc www-web-0 www-web-1 lab-data
kubectl delete pod cache-demo
rm -f emptydir-pod.yaml data-pvc.yaml writer-pod.yaml web-stateful.yaml snap.yaml bad-pvc.yaml
```

**Note:** `kubectl delete pvc` with a Delete reclaim policy destroys the underlying volume — exactly why Exercise 4's reclaim policy question matters before you run this in production.
