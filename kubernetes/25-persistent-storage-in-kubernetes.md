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
