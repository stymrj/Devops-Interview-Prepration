# Persistent Storage in Kubernetes — Cheat Sheet

## Core Objects

| Object | What it is | Who creates it |
|---|---|---|
| PersistentVolume (PV) | Actual chunk of storage in the cluster | Admin, or StorageClass dynamically |
| PersistentVolumeClaim (PVC) | Request: "I need X GB, mode Y" | You (dev) |
| StorageClass | Template for dynamic provisioning (provisioner + params) | Admin |
| VolumeSnapshot | Point-in-time copy of a volume | You |

## Access Modes

| Mode | Meaning | EBS? |
|---|---|---|
| `ReadWriteOnce` (RWO) | One node, read-write | Yes |
| `ReadOnlyMany` (ROX) | Many nodes, read-only | Yes |
| `ReadWriteMany` (RWX) | Many nodes, read-write | **No** — use EFS/NFS |

## Reclaim Policies

| Policy | On PVC delete |
|---|---|
| `Retain` | Volume + data survive, PV goes `Released` |
| `Delete` | Volume **and data destroyed** (default for dynamic) |
| `Recycle` | Deprecated, ignore |

## Volume Types (speed of recall)

- `emptyDir` — Pod lifetime, dies with Pod. Scratch/cache.
- `hostPath` — node's disk, dies with node. **Avoid in prod.**
- `persistentVolumeClaim` — network storage (EBS/EFS). Real state.
- `configMap` / `secret` — config as files (read-only).
- `local` — node's disk but scheduler-aware (needs node affinity).

## Essential Commands

```bash
kubectl get pv,pvc                              # list storage
kubectl describe pvc <name>                     # why Pending? read Events
kubectl get storageclass                         # available provisioners
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'  # expand
kubectl get volumesnapshotclass                   # snapshot support?
kubectl delete pvc <name>                        # destroys volume if reclaim=Delete!
```

## PV ↔ PVC Binding Rules

- Capacity: PV ≥ PVC request
- Access modes: must intersect
- StorageClass names must match (or both empty)
- PVC is namespaced, PV is cluster-scoped
- A Pod references the **PVC**, never the PV

## StatefulSet Storage Essentials

```yaml
volumeClaimTemplates:          # one PVC per replica: data-web-0, data-web-1...
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests: { storage: 20Gi }
```

- PVCs survive Pod deletion **and** scale-down — delete them manually
- Pods get stable names + ordered startup/shutdown
- CSI driver name in StorageClass tells you who actually provides storage (e.g. `ebs.csi.aws.com`)

## Interview Traps to Remember

1. Deleting a dynamically-provisioned PVC **deletes the data** (reclaim=Delete)
2. Env-var injection is frozen at start; volume mounts can update
3. EBS can never be ReadWriteMany
4. Volumes only grow, never shrink — and only with `allowVolumeExpansion: true`
5. `subPath` mounts don't refresh when the source changes
6. PVC stuck Pending → `kubectl describe pvc`, read the Events
7. EBS volumes can't leave their AZ; regional storage (EFS) can
