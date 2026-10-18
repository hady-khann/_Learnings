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

Important: the Ceph cluster and the Kubernetes cluster are **separate**. Kubernetes connects to Ceph over the network. Installing Ceph-CSI does not redeploy Ceph or touch an existing pool.

---

# 2. Create a Ceph RBD Pool

For a simple test environment:

```bash
ceph osd pool create rbd-pool-app1 50

ceph osd pool set rbd-pool-app1 size 3

ceph osd pool application enable rbd-pool-app1 rbd

rbd pool init rbd-pool-app1
```

`rbd pool init` prepares the pool for RBD use. It does not delete existing data.

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

Clean up the manual test before using the pool with Kubernetes:

```bash
umount /mnt/rbd-mount-app1
rbd unmap /dev/rbd0
rbd rm rbd-pool-app1/rbd-image-app1
```

This manual workflow is useful for learning RBD, but Kubernetes normally performs these operations through Ceph-CSI.

---

# 4. Prepare Ceph for Kubernetes

Run these on a Ceph admin node.

## 4.1 Create a dedicated Ceph user

Do not use `client.admin` for Kubernetes. Create a user limited to the pool:

```bash
ceph auth get-or-create client.kubernetes \
  mon 'profile rbd' \
  osd 'profile rbd pool=rbd-pool-app1' \
  mgr 'profile rbd pool=rbd-pool-app1'
```

Get the key:

```bash
ceph auth get-key client.kubernetes
```

Store the key securely and never commit it to Git.

## 4.2 Collect cluster connection details

```bash
ceph fsid

ceph mon dump
```

Example (illustrative only):

```text
FSID:     12345678-1234-1234-1234-123456789abc
MONITORS: 192.168.1.10:6789, 192.168.1.11:6789, 192.168.1.12:6789
```

If your monitors use msgr2 only, use the ports shown by `ceph mon dump` (for example `3300`) in the configuration.

## 4.3 Network requirements

Every Kubernetes worker node that may mount a volume must reach:

* All Ceph monitors (ports 3300 and/or 6789)
* All OSDs (ports 6800–7300)

Test from a worker node:

```bash
nc -zv <MONITOR_IP> 6789
```

Worker nodes also need the `rbd` kernel module available:

```bash
modprobe rbd
lsmod | grep rbd
```

---

# 5. Install Ceph-CSI in Kubernetes

Check whether a driver is already installed:

```bash
kubectl get csidrivers

kubectl get pods -A | grep -i ceph

helm list -A | grep -i ceph
```

If `rbd.csi.ceph.com` already exists and its Pods are healthy, skip the install.

## 5.1 Prerequisite for snapshots

VolumeSnapshots need the snapshot CRDs and the snapshot controller. Check:

```bash
kubectl get crd | grep snapshot.storage.k8s.io

kubectl get pods -A | grep snapshot-controller
```

If they are missing, install them from the `kubernetes-csi/external-snapshotter` project (many distributions already include them).

## 5.2 Install with Helm

Create a `values.yaml`:

```yaml
csiConfig:
  - clusterID: "<YOUR_CEPH_FSID>"
    monitors:
      - "<MONITOR_1_IP>:6789"
      - "<MONITOR_2_IP>:6789"
      - "<MONITOR_3_IP>:6789"
```

Install:

```bash
helm repo add ceph-csi https://ceph.github.io/csi-charts

helm repo update

helm install ceph-csi-rbd ceph-csi/ceph-csi-rbd \
  --namespace ceph-csi-rbd \
  --create-namespace \
  -f values.yaml
```

Check:

```bash
kubectl -n ceph-csi-rbd get pods

kubectl get csidrivers
```

Expected: a `provisioner` Deployment, a `nodeplugin` DaemonSet (one Pod per node), and the `rbd.csi.ceph.com` CSIDriver object.

## 5.3 Alternative: Ceph-CSI Operator

The Ceph-CSI Operator is the newer way to manage the driver and is documented upstream. Its resources differ (`Driver`, `ClientProfile`, and others), and when using it the StorageClass `clusterID` must match the configured **ClientProfile name** instead of the Ceph FSID. Check the operator's documentation for your Kubernetes version before choosing it. The rest of this guide uses the Helm install, where `clusterID` is the Ceph FSID.

---

# 6. Create the Secret

The Secret holds the Ceph user and key. Create it in the same namespace used in the StorageClass:

```bash
kubectl -n ceph-csi-rbd create secret generic csi-rbd-secret \
  --from-literal=userID=kubernetes \
  --from-literal=userKey='<YOUR_CEPH_CLIENT_KEY>'
```

Check:

```bash
kubectl -n ceph-csi-rbd get secret csi-rbd-secret
```

Note: `userID` is `kubernetes`, **without** the `client.` prefix.

---

# 7. Ceph-CSI StorageClass

A complete RBD StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass

metadata:
  name: ceph-rbd

provisioner: rbd.csi.ceph.com

parameters:
  clusterID: "<YOUR_CEPH_FSID>"
  pool: rbd-pool-app1

  imageFeatures: layering

  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-rbd
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi-rbd
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-rbd

  csi.storage.k8s.io/fstype: ext4

reclaimPolicy: Delete

allowVolumeExpansion: true

volumeBindingMode: Immediate
```

Apply:

```bash
kubectl apply -f storageclass.yaml

kubectl get storageclass
```

Key options:

| Option | Meaning |
| --- | --- |
| `clusterID` | Must match an entry in the Ceph-CSI config (`csiConfig`) |
| `pool` | The Ceph pool where RBD images are created |
| `imageFeatures` | `layering` is the safe default for kernel-mapped images |
| `*-secret-*` | Credentials used for provisioning, expanding and mounting |
| `fstype` | Filesystem created on the RBD (`ext4` or `xfs`) |
| `reclaimPolicy` | `Delete` removes the RBD image with the PV; `Retain` keeps it |
| `allowVolumeExpansion` | Required for PVC resize |

The important option for resize is:

```yaml
allowVolumeExpansion: true
```

For production, consider `reclaimPolicy: Retain` and `volumeBindingMode: WaitForFirstConsumer`.

---

# 8. Create a PVC

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

Confirm on the Ceph side:

```bash
rbd ls -p rbd-pool-app1
```

You should see an image named like `csi-vol-<uuid>`.

---

# 9. Use the PVC from a Pod

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

Test persistence:

```bash
kubectl exec app1 -- sh -c 'echo "hello ceph" > /data/test.txt'

kubectl delete pod app1
kubectl apply -f pod.yaml

kubectl exec app1 -- cat /data/test.txt
```

The file should still be there after the Pod is recreated.

---

# 10. Resize the RBD Volume

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

Look at the PVC conditions if the size does not change:

```bash
kubectl describe pvc app1-pvc
```

Then check from the Pod:

```bash
kubectl exec -it app1 -- df -h
```

Notes:

* Volumes can only be **grown**, never shrunk.
* RBD volumes support online expansion; the filesystem is grown automatically while the Pod is running (a Pod restart may be needed on some older versions).

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

# 11. Direct RBD Resize vs Kubernetes Resize

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

Prefer the Kubernetes method when Kubernetes/CSI owns the Volume. A direct `rbd resize` leaves the PVC/PV size out of sync with the real image.

---

# 12. VolumeSnapshotClass

Before creating Kubernetes VolumeSnapshots, a suitable `VolumeSnapshotClass` must exist, along with the snapshot CRDs and controller (see section 5.1).

Example:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass

metadata:
  name: ceph-rbd-snapshotclass

driver: rbd.csi.ceph.com

parameters:
  clusterID: "<YOUR_CEPH_FSID>"
  csi.storage.k8s.io/snapshotter-secret-name: csi-rbd-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: ceph-csi-rbd

deletionPolicy: Delete
```

Apply and check:

```bash
kubectl apply -f snapshotclass.yaml

kubectl get volumesnapshotclass
```

The `clusterID` and secret must match the StorageClass configuration.

---

# 13. Create a VolumeSnapshot

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

`READYTOUSE` should become `true`.

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

# 14. Direct RBD Snapshot

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
1       before-upgrade   15 GiB
```

---

# 15. Restore from a Kubernetes Snapshot

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

The requested size must be equal to or larger than the snapshot's source size.

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

To use the restored data, point the application (or a new Pod) at `app1-pvc-restored`.

---

# 16. Clone a PVC

A PVC can also be cloned directly, without creating a snapshot first:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: app1-pvc-clone

spec:
  storageClassName: ceph-rbd

  dataSource:
    name: app1-pvc
    kind: PersistentVolumeClaim

  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 15Gi
```

Apply and check:

```bash
kubectl apply -f clone-pvc.yaml

kubectl get pvc
```

The clone must be in the same namespace and use the same StorageClass as the source PVC.

---

# 17. Direct RBD Rollback

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

# 18. Rollback vs Restore

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

# 19. Check the RBD Backend

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

Check which clients have an image open:

```bash
rbd status rbd-pool-app1/<image-name>
```

Check pool:

```bash
ceph osd pool get rbd-pool-app1 size
```

Map a PV to its RBD image:

```bash
kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeAttributes.imageName}'
```

---

# 20. Troubleshooting

### PVC stays in `Pending`

```bash
kubectl describe pvc app1-pvc

kubectl -n ceph-csi-rbd get pods

kubectl -n ceph-csi-rbd logs deploy/ceph-csi-rbd-provisioner -c csi-rbdplugin
```

Common causes:

* Wrong `clusterID` (does not match the `csiConfig` entry)
* Wrong Secret name/namespace, or wrong key
* Monitors unreachable from the provisioner Pods
* Ceph user lacks permissions on the pool
* Typo in the StorageClass name

### Pod stuck in `ContainerCreating`

```bash
kubectl describe pod app1

kubectl -n ceph-csi-rbd logs ds/ceph-csi-rbd-nodeplugin -c csi-rbdplugin
```

Common causes:

* Worker node cannot reach the OSDs
* `rbd` kernel module missing on the node
* Image still mapped on another node (RWO volume, previous Pod not fully terminated)
* Unsupported `imageFeatures` for the node's kernel

### Resize does not complete

```bash
kubectl describe pvc app1-pvc
```

Check `allowVolumeExpansion: true`, the `controller-expand-secret-*` parameters, and the provisioner logs.

### Snapshot is not `ReadyToUse`

```bash
kubectl describe volumesnapshot app1-snapshot

kubectl describe volumesnapshotcontent
```

Check the snapshot controller is running, the `VolumeSnapshotClass` has the correct secret, and the `clusterID` matches.

### Ceph health

```bash
ceph -s

ceph health detail
```

---

# 21. Important Kubernetes Commands

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

CSI driver:

```bash
kubectl get csidrivers
kubectl -n ceph-csi-rbd get pods
```

Filesystem:

```bash
kubectl exec -it app1 -- df -h
```

---

# 22. Clean Up

Delete in this order:

```bash
kubectl delete pod app1

kubectl delete volumesnapshot app1-snapshot

kubectl delete pvc app1-pvc-restored app1-pvc-clone app1-pvc
```

With `reclaimPolicy: Delete`, deleting the PVC also deletes the PV and the RBD image. Verify:

```bash
kubectl get pv

rbd ls -p rbd-pool-app1
```

With `reclaimPolicy: Retain`, the PV and RBD image stay and must be removed manually.

---

# 23. Important Rules

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

### Rule 6 — Use a dedicated Ceph user

Never put `client.admin` credentials in a Kubernetes Secret. Use a user restricted to the required pool.

---

### Rule 7 — Do not manually edit CSI-managed images

Do not rename, delete or modify `csi-vol-*` images by hand while a PV references them.

---

# 24. RWO vs Shared Access

RBD is block storage. A normal filesystem-mode RBD volume is:

```yaml
accessModes:
  - ReadWriteOnce
```

This means one node at a time. It is not suitable for multiple Pods on different nodes writing to the same filesystem.

| Need | Use |
| --- | --- |
| One Pod / one node, databases, single-instance apps | RBD, `ReadWriteOnce` |
| Many Pods sharing files (RWX) | CephFS (Ceph-CSI CephFS driver) or NFS |
| Raw shared block device for clustered software | RBD with `volumeMode: Block` and `ReadWriteMany` (advanced, the application must handle concurrent access safely) |

Multiple Pods on the **same node** can use an RWO volume, but do not rely on this for scaling.

---

# 25. Final Mental Model

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

Setup order for a new cluster:

1. Create and initialize the Ceph pool
2. Create a restricted Ceph user
3. Collect FSID and monitor addresses
4. Install Ceph-CSI (and snapshot CRDs/controller)
5. Create the Secret
6. Create the StorageClass
7. Test with a PVC and Pod
8. Add VolumeSnapshotClass and test snapshot/restore

For Kubernetes-managed RBD volumes, the most important concepts to learn are:

1. **StorageClass**
2. **PVC / PV**
3. **Ceph-CSI**
4. **Volume Expansion**
5. **VolumeSnapshot**
6. **Restore / Clone**
7. **RBD Pool / Image**
8. **RWO vs shared filesystem requirements**
