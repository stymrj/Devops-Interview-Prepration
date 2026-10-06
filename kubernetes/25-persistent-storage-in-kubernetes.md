# Persistent Storage in Kubernetes Interview Preparation Guide

*How to Answer Kubernetes Storage Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Storage Basics Volumes PVs and PVCs](#storage-basics-volumes-pvs-and-pvcs)
2. [StorageClasses and Dynamic Provisioning](#storageclasses-and-dynamic-provisioning)
3. [StatefulSets and Data Patterns](#statefulsets-and-data-patterns)
4. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

---

## Storage Basics Volumes PVs and PVCs

### Q1: Why does a Pod lose its data when it restarts, and what fixes it?

**How to Answer:**

"A container's writable filesystem is tied to the container's life, not the Pod's. When the container restarts — new image, crash, OOMKill — everything it wrote is gone. The Pod can even be rescheduled to another node, and then the new node has none of the files."

"The fix is a volume. A volume outlives the container inside the Pod and, depending on the type, outlives the Pod entirely. For anything stateful, you need persistent storage — something like a PersistentVolume backed by EBS or EFS — so data survives restarts, rollouts, and node moves."

**Key Point:** "Container filesystems die with the container — volumes are what make data survive."

---

### Q2: What's the difference between a PersistentVolume and a PersistentVolumeClaim?

**How to Answer:**

"A PersistentVolume is the actual chunk of storage in the cluster — say a 50GB EBS volume — usually created by an admin or by a StorageClass. A PersistentVolumeClaim is the request: 'I need 50GB, ReadWriteOnce.' Kubernetes binds the claim to a matching volume, and then a Pod uses the claim."

"The separation exists so developers request storage without caring about the underlying disk. I define what I need in the PVC, the cluster figures out which PV satisfies it. A Pod never references a PV directly — it mounts the PVC."

```yaml
volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mysql-data
```

**Key Point:** "PV is the resource, PVC is the request — Pods only ever touch the claim."

---

### Q3: What are the access modes, and why does ReadWriteMany always come up?

**How to Answer:**

"There are three: ReadWriteOnce means one node can mount it read-write, ReadOnlyMany means many nodes can read, and ReadWriteMany means many nodes can mount it read-write at the same time. ReadWriteOnce is the default for block storage like EBS."

"ReadWriteMany comes up because most databases and apps only need RWO, but shared content — media uploads, shared caches — needs RWX. The trap is that EBS physically can't do RWX. If an interviewer asks for it, the answer is a file system like EFS or NFS, not EBS."

**Key Point:** "RWO is one node, RWX is many — and block storage like EBS can never do RWX."

---

### Q4: What happens to my data when I delete the PVC?

**How to Answer:**

"That depends on the PV's reclaim policy. With Retain, the volume and its data stay — Kubernetes just marks it Released and nobody else can claim it until an admin clears it. With Delete, the volume and the data are destroyed. Recycle barely exists anymore."

"The nasty default: dynamically provisioned volumes get Delete. So if you delete a PVC by accident, your storage and your data vanish. That's why I always set Retain on anything important — databases, anything I can't rebuild — and let the cluster garbage-collect the throwaway stuff."

**Key Point:** "Reclaim policy decides data's fate — Delete is the default for dynamic volumes, so use Retain for anything precious."

---

### Q5: Can I resize a volume, or do I have to create a bigger one?

**How to Answer:**

"You can resize in place, but the StorageClass has to allow it — it needs `allowVolumeExpansion: true`. Then you just edit the PVC and bump the storage request, and the volume grows. The PV and the underlying disk get updated, and you usually expand the filesystem too."

"The gotcha is you can only grow, never shrink. And some drivers need a Pod restart to finish the filesystem resize, so a Pod restart after expanding is normal. Shrinking means migrating the data to a new volume — there's no shortcut."

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
parameters:
  type: gp3
```

**Key Point:** "Volumes only grow, never shrink — and only if the StorageClass allows expansion."

---

### Q6: What's the difference between emptyDir, hostPath, and a real persistent volume?

**How to Answer:**

"emptyDir is a directory on the node's disk that lives as long as the Pod does — container restarts survive, but if the Pod is deleted or rescheduled, the data is gone. It's a scratch space: caches, temp files, shared workspace between containers in a Pod."

"hostPath mounts a directory from the node's own filesystem, so it survives the Pod but it's pinned to one node — move the Pod and the data stays behind. A real persistent volume like EBS is network storage: it follows the Pod to any node in the zone, survives Pod deletion with the right reclaim policy, and is the only sane choice for actual state."

**Key Point:** "emptyDir dies with the Pod, hostPath dies with the node, network volumes follow the data wherever it goes."

---

### Q7: How does storage actually follow the Pod from node to node?

**How to Answer:**

"When a Pod with a PVC gets rescheduled, the scheduler knows the bound PV lives in a specific availability zone. It only places the Pod on a node in that zone, and the storage driver's attach/detach controllers unmount the volume from the old node and attach it to the new one. That's why an EBS-backed Pod can move nodes but never zones."

"If you need volumes that can move zones, you need replicated storage — a driver or storage system that mirrors data across zones, like EFS which is regional by default, or a distributed store like Ceph. Single-zone disks physically can't follow a Pod across zones."

**Key Point:** "Volumes follow Pods within their zone; cross-zone mobility needs replicated storage, not a bigger disk."

---
## StatefulSets and Data Patterns

### Q8: Why can't I just run a database in a Deployment?

**How to Answer:**

"A Deployment gives every replica the same identity and the same storage — Pods are interchangeable and interchangeable storage is fine for stateless apps. A database replica isn't interchangeable: each one owns its own data, its own replication role, and needs a stable name its peers can find."

"That's what StatefulSets fix: stable pod names, ordered rollout and scaling, and each replica gets its own persistent storage that sticks to it. I use Deployments for stateless and StatefulSets for anything with data — databases, queues, anything with peer discovery."

**Key Point:** "Databases need stable identity and per-replica storage — that's StatefulSet territory, not Deployments."

---

### Q9: How does volumeClaimTemplates work in a StatefulSet?

**How to Answer:**

"You declare the PVC template once in the StatefulSet spec, and Kubernetes creates one PVC per replica — `data-myapp-0`, `data-myapp-1`, and so on. Each PVC is bound to its own PV, so every replica has its private disk."

"The killer feature: the PVC sticks around when the Pod dies. If `myapp-1` crashes and comes back on a different node, it reclaims `data-myapp-1` and its data. Even scaling the StatefulSet down doesn't delete the PVCs — they have to be deleted manually, which is annoying but protects you from wiping data with one command."

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

**Key Point:** "One template, one PVC per replica — and the data outlives the Pod."

---

### Q10: Local storage versus network storage — what's the real tradeoff?

**How to Answer:**

"Local storage — local PVs, node disks — is fast and cheap because there's no network hop. The tradeoff is the data is married to that node: if the node dies, the data is gone unless you replicated it yourself. Network storage like EBS survives node failure but adds latency and cost."

"So the rule: use local for things that can rebuild — caches, build artifacts, throwaway compute — and network storage for anything you'd miss. Also, local volumes need node affinity so the scheduler keeps the Pod on the right node; otherwise the Pod lands elsewhere and can't find its disk."

**Key Point:** "Local is fast but dies with the node; network is slower but the data survives."

---

### Q11: What is a CSI driver and why does it matter in practice?

**How to Answer:**

"CSI — the Container Storage Interface — is the standard plugin system that lets storage vendors hook into Kubernetes. The EBS driver, the EFS driver, Portworx, Longhorn — they're all CSI drivers. Kubernetes talks to them through a common interface for attach, detach, mount, snapshots, and expansion."

"Why it matters: in-tree volume plugins are frozen and deprecated, so all new storage features land in CSI drivers. If your cluster needs snapshots, cloning, or volume expansion, that's the CSI driver providing it. When debugging storage issues, the driver's logs and its provisioner name in the StorageClass are usually where the answer is."

**Key Point:** "CSI drivers are how real storage plugs into Kubernetes — every modern storage feature flows through them."

---

### Q12: How do volume snapshots work, and when would I use them?

**How to Answer:**

"A snapshot is a point-in-time copy of a volume's data, managed by the CSI driver — same idea as an EBS snapshot. You create a VolumeSnapshot resource, and later you can create a brand-new PVC from that snapshot instead of from scratch. The new volume starts life containing the snapshot's data."

"I use snapshots for backups before risky migrations and for cloning: spin up a staging database from a production snapshot without touching production. They're not a disaster recovery plan on their own — you still need retention and off-cluster copies — but for quick restores they're the fastest tool available."

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-backup
spec:
  volumeSnapshotClassName: ebs-snapshot
  source:
    persistentVolumeClaimName: mysql-data
```

**Key Point:** "Snapshots are point-in-time volume copies — restores and clones both start from them."

---

### Q13: What's the subPath trap with mounted volumes?

**How to Answer:**

"subPath lets you mount one file or directory from inside a volume instead of the whole thing — like mounting just `app.conf` from a ConfigMap without hiding the rest of the directory. It's handy, but mounted files via subPath don't update when the underlying ConfigMap or Secret changes."

"So the trap: you use subPath to inject a single config file, the config gets updated, and your Pod never sees the new value while every other Pod does. I avoid subPath unless I truly need to mount a single file over an existing directory, and even then I'd rather restructure the mount so the whole volume is used."

**Key Point:** "subPath mounts go stale on updates — mount whole volumes when config needs to stay fresh."

---

## Troubleshooting and Best Practices

### Q14: My PVC is stuck in Pending. How do I debug it?

**How to Answer:**

"First, `kubectl describe pvc` and read the Events — Kubernetes is usually honest about why. The classics: no StorageClass on the cluster or a typo in the name, no PV matching the requested size and access mode, the driver couldn't create the volume, or the Pod is in a different zone than the volume."

"My flow is describe the PVC, then describe the StorageClass, then check the PVs. If it's dynamic provisioning, I look at the provisioner's logs — the external-provisioner container usually names the exact API failure. If it's static, I check whether a free PV with matching capacity, access mode, and StorageClass actually exists."

```bash
kubectl describe pvc mysql-data
kubectl get storageclass
kubectl get pv | grep -i released
```

**Key Point:** "A Pending PVC is a matching problem — describe the PVC, the StorageClass, and the PVs in that order."

---

*Part of the [DevOps Interview Preparation](../README.md) series — Topic 25 of 58.*
