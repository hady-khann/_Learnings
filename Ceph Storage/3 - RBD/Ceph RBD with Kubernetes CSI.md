# Ceph RBD with Kubernetes CSI

## 1. Overview

When Ceph RBD is used with Kubernetes, **Ceph-CSI** acts as the bridge between Kubernetes and Ceph.

```text
Pod
 │
 ▼
PVC
 │
 ▼
PV
 │
 ▼
Ceph-CSI
 │
 ▼
Ceph RBD Image
 │
 ▼
Ceph Pool
 │
 ▼
PGs → OSDs
```

Ceph-CSI handles:

* RBD provisioning
* Mapping and mounting
* Volume expansion
* Snapshots
* Cloning
* Restore

---

# 2. Create a Ceph RBD Pool

For a simple test environment:

```bash
ceph osd pool create rbd-pool-app1 50

ceph osd pool set rbd-pool-app1 size 3

ceph osd pool application enable rbd-pool-app1 rbd
```

Check:

```bash
ceph osd pool ls

ceph osd pool get rbd-pool-app1 size

ceph osd pool application get rbd-pool-app1
```

> In production, PG numbers should normally be planned or managed using the Ceph PG autoscaler rather than choosing an arbitrary value.

---

# 3. Create an RBD Image Manually

This is useful for understanding RBD itself:

```bash
rbd create rbd-image-app1 \
  --pool rbd-pool-app1 \
  --size 10G
```

Check:

```bash
rbd info rbd-pool-app1/rbd-image-app1
```

Map it:

```bash
rbd map rbd-pool-app1/rbd-image-app1
```

Check:

```bash
rbd showmapped

lsblk
```

Create a filesystem:

```bash
mkfs.xfs /dev/rbd0
```

Mount:

```bash
mkdir -p /mnt/rbd-mount-app1

mount /dev/rbd0 /mnt/rbd-mount-app1
```

Check:

```bash
df -h
df -Th
```

This manual workflow is useful for learning RBD, but Kubernetes normally performs these operations through Ceph-CSI.

---

# 4. Ceph-CSI StorageClass

A simplified RBD StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: ceph-rbd

provisioner: rbd.csi.ceph.com

parameters:
  clusterID: <CEPH_CLUSTER_ID>
  pool: rbd-pool-app1

  imageFeatures: layering

reclaimPolicy: Delete

allowVolumeExpansion: true

volumeBindingMode: Immediate
```

The important option for resize is:

```yaml
allowVolumeExpansion: true
```

In a real environment, the StorageClass also needs the appropriate Ceph-CSI authentication/secret configuration.

---

# 5. Create a PVC

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: app1-pvc

spec:
  storageClassName: ceph-rbd

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Check:

```bash
kubectl get pvc
```

Expected:

```text
NAME        STATUS   VOLUME     CAPACITY
app1-pvc    Bound    pvc-xxxx   10Gi
```

---

# 6. Use the PVC from a Pod

Example Pod:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: app1

spec:
  containers:
    - name: app
      image: nginx

      volumeMounts:
        - name: app-storage
          mountPath: /data

  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: app1-pvc
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pod app1
```

Check the filesystem:

```bash
kubectl exec -it app1 -- df -h
```

---

# 7. Resize the RBD Volume

Assume:

```text
Current size = 10Gi
Target size  = 15Gi
```

Do **not** normally resize the RBD directly with:

```bash
rbd resize ...
```

Instead resize the PVC.

```bash
kubectl patch pvc app1-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"15Gi"}}}}'
```

Or:

```bash
kubectl edit pvc app1-pvc
```

Change:

```yaml
resources:
  requests:
    storage: 15Gi
```

Check:

```bash
kubectl get pvc app1-pvc
```

Then check from the Pod:

```bash
kubectl exec -it app1 -- df -h
```

The expected flow is:

```text
PVC 10Gi
   ↓
PVC 15Gi
   ↓
Ceph-CSI
   ↓
RBD 15Gi
   ↓
Filesystem 15Gi
```

---

# 8. Direct RBD Resize vs Kubernetes Resize

### Direct RBD

```bash
rbd resize \
  --pool rbd-pool-app1 \
  --image rbd-image-app1 \
  --size 15G
```

For XFS:

```bash
xfs_growfs -d /mount/path
```

For EXT4:

```bash
resize2fs /dev/rbd0
```

### Kubernetes

```bash
kubectl patch pvc app1-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"15Gi"}}}}'
```

Prefer the Kubernetes method when Kubernetes/CSI owns the Volume.

---

# 9. VolumeSnapshotClass

Before creating Kubernetes VolumeSnapshots, a suitable `VolumeSnapshotClass` must exist.

Example:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass

metadata:
  name: ceph-rbd-snapshotclass

driver: rbd.csi.ceph.com

deletionPolicy: Delete
```

Check:

```bash
kubectl get volumesnapshotclass
```

The exact configuration depends on the Ceph-CSI deployment and authentication setup.

---

# 10. Create a VolumeSnapshot

Example:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot

metadata:
  name: app1-snapshot

spec:
  volumeSnapshotClassName: ceph-rbd-snapshotclass

  source:
    persistentVolumeClaimName: app1-pvc
```

Apply:

```bash
kubectl apply -f snapshot.yaml
```

Check:

```bash
kubectl get volumesnapshot
```

More details:

```bash
kubectl describe volumesnapshot app1-snapshot
```

Architecture:

```text
PVC
 │
 ▼
VolumeSnapshot
 │
 ▼
Ceph-CSI
 │
 ▼
RBD Snapshot
```

---

# 11. Direct RBD Snapshot

If managing RBD directly:

```bash
rbd snap create \
  rbd-pool-app1/rbd-image-app1@before-upgrade
```

List snapshots:

```bash
rbd snap ls rbd-pool-app1/rbd-image-app1
```

Example:

```text
SNAPID  NAME              SIZE
1       before-upgrade   15Gi
```

---

# 12. Restore from a Kubernetes Snapshot

A common Kubernetes workflow is to create a **new PVC from the Snapshot**.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: app1-pvc-restored

spec:
  storageClassName: ceph-rbd

  dataSource:
    name: app1-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 15Gi
```

Apply:

```bash
kubectl apply -f restored-pvc.yaml
```

Check:

```bash
kubectl get pvc
```

Flow:

```text
Original PVC
     │
     ▼
VolumeSnapshot
     │
     ▼
New PVC
     │
     ▼
New RBD Image
```

The original RBD remains unchanged.

---

# 13. Direct RBD Rollback

With manually managed RBD:

```bash
rbd snap rollback \
  rbd-pool-app1/rbd-image-app1@before-upgrade
```

This changes the **existing RBD** back to the Snapshot state.

```text
Existing RBD
     │
     ▼
Rollback
     │
     ▼
Old state
```

Be careful when the RBD is mounted or actively used.

Do not casually perform direct RBD rollback on a volume that is actively being used by a Kubernetes Pod.

---

# 14. Rollback vs Restore

### RBD Rollback

```bash
rbd snap rollback POOL/IMAGE@SNAPSHOT
```

Changes the existing RBD.

```text
Existing RBD
     ↓
Previous state
```

### Kubernetes Restore

```text
VolumeSnapshot
      ↓
New PVC
      ↓
New RBD
```

The original PVC/RBD remains unchanged.

For Kubernetes workloads, the second approach is generally easier to control.

---

# 15. Check the RBD Backend

Even when Kubernetes manages the Volume, RBD commands are useful for troubleshooting.

List images:

```bash
rbd ls -p rbd-pool-app1
```

Show image information:

```bash
rbd info rbd-pool-app1/<image-name>
```

List snapshots:

```bash
rbd snap ls rbd-pool-app1/<image-name>
```

Check pool:

```bash
ceph osd pool get rbd-pool-app1 size
```

---

# 16. Important Kubernetes Commands

PVC:

```bash
kubectl get pvc
kubectl describe pvc app1-pvc
```

PV:

```bash
kubectl get pv
kubectl describe pv <pv-name>
```

StorageClass:

```bash
kubectl get storageclass
kubectl describe storageclass ceph-rbd
```

Snapshots:

```bash
kubectl get volumesnapshot
kubectl describe volumesnapshot app1-snapshot
```

Pods:

```bash
kubectl get pods
kubectl describe pod app1
```

Filesystem:

```bash
kubectl exec -it app1 -- df -h
```

---

# 17. Important Rules

### Rule 1 — Kubernetes owns Kubernetes Volumes

If the RBD was provisioned by Ceph-CSI:

```text
Kubernetes
    ↓
PVC
    ↓
Ceph-CSI
    ↓
RBD
```

Prefer Kubernetes APIs for lifecycle operations.

---

### Rule 2 — Resize the PVC

Prefer:

```bash
kubectl patch pvc ...
```

instead of:

```bash
rbd resize ...
```

---

### Rule 3 — Use VolumeSnapshot

Prefer:

```text
VolumeSnapshot
```

instead of manually creating RBD snapshots for Kubernetes-managed volumes.

---

### Rule 4 — Be careful with direct rollback

Avoid:

```bash
rbd snap rollback ...
```

on an actively mounted Kubernetes volume.

---

### Rule 5 — RBD is Block Storage

RBD itself does not care whether the filesystem is:

```text
XFS
EXT4
```

The filesystem exists above the RBD block device.

---

# 18. Final Mental Model

```text
                         Kubernetes
                              │
                              ▼
                             Pod
                              │
                              ▼
                             PVC
                              │
                              ▼
                             PV
                              │
                              ▼
                          Ceph-CSI
                              │
                ┌─────────────┼─────────────┐
                │             │             │
              Create        Resize       Snapshot
                │             │             │
                └─────────────┼─────────────┘
                              ▼
                          Ceph RBD
                              │
                              ▼
                             Pool
                              │
                              ▼
                         PGs / CRUSH
                              │
                              ▼
                            OSDs
```

The key concept is:

> **RBD provides the block storage, Ceph-CSI connects it to Kubernetes, and Kubernetes manages the Volume lifecycle through PVCs and CSI APIs.**

For Kubernetes-managed RBD volumes, the most important concepts to learn are:

1. **StorageClass**
2. **PVC / PV**
3. **Ceph-CSI**
4. **Volume Expansion**
5. **VolumeSnapshot**
6. **Restore / Clone**
7. **RBD Pool / Image**
8. **RWO vs shared filesystem requirements**
