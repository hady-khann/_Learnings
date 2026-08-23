# Ceph RBD: Snapshot, Unmap, Rollback and Remount

## Overview

An RBD snapshot provides a point-in-time state of an RBD image.

Typical workflow:

``` text
Mounted RBD
    │
    ▼
Create Snapshot
    │
    ▼
Perform changes
    │
    ▼
Unmount / Unmap
    │
    ▼
Rollback
    │
    ▼
Map / Mount again
```

## 1. Create an RBD Snapshot

Create a snapshot:

``` bash
rbd snap create \
  rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823
```

List snapshots:

``` bash
rbd snap ls rbd-pool-app1/rbd-image-app1
```

Example:

``` text
SNAPID  NAME
1       rbd-image-app1-20260823
```

The snapshot name follows:

``` text
POOL/IMAGE@SNAPSHOT
```

## 2. Why Use a Snapshot?

Snapshots are useful for:

-   Creating a point-in-time recovery point
-   Taking a snapshot before an upgrade
-   Testing changes
-   Creating clones from a known state

Example:

``` text
Before upgrade
      │
      ▼
Create Snapshot
      │
      ▼
Upgrade application
      │
      ▼
Something goes wrong
      │
      ▼
Rollback
```

## 3. Stop Using the Filesystem Before Rollback

Before rolling back an RBD that is mounted, stop applications using it
and unmount the filesystem:

``` bash
umount /mnt/rbd-mount-app1
```

Then unmap the RBD:

``` bash
rbd unmap /dev/rbd0
```

Verify:

``` bash
rbd showmapped
```

The image should no longer appear as a mapped device.

> Do not casually rollback an RBD while its filesystem is actively being
> used. The application and filesystem should be in a controlled state
> before recovery.

## 4. Rollback the RBD

Rollback to the snapshot:

``` bash
rbd snap rollback \
  rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823
```

This changes the existing RBD image back to the state represented by the
snapshot.

Conceptually:

``` text
Current RBD
    │
    │ rollback
    ▼
Snapshot state
```

Data written after the snapshot is no longer represented in the current
image state.

## 5. Map the RBD Again

After rollback:

``` bash
rbd map rbd-pool-app1/rbd-image-app1
```

Check:

``` bash
rbd showmapped
```

You should again see something similar to:

``` text
/dev/rbd0
```

## 6. Mount the Filesystem

Mount it again:

``` bash
mount /dev/rbd0 /mnt/rbd-mount-app1
```

Verify:

``` bash
df -h /mnt/rbd-mount-app1
df -Th
```

## 7. Complete Workflow

``` bash
# Create snapshot
rbd snap create \
  rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823

# List snapshots
rbd snap ls rbd-pool-app1/rbd-image-app1

# Stop applications using the filesystem
# Then unmount
umount /mnt/rbd-mount-app1

# Unmap
rbd unmap /dev/rbd0

# Rollback
rbd snap rollback \
  rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823

# Map again
rbd map rbd-pool-app1/rbd-image-app1

# Mount again
mount /dev/rbd0 /mnt/rbd-mount-app1

# Verify
df -h /mnt/rbd-mount-app1
```

## 8. Snapshot vs Backup

An RBD snapshot is not automatically a full backup.

The snapshot remains inside the same Ceph environment:

``` text
Ceph Cluster
   │
   ├── RBD Image
   │
   └── RBD Snapshot
```

If the entire Ceph cluster is lost, the snapshot may be lost as well.

For disaster recovery, use an appropriate backup or replication
strategy.

## 9. Important Warning

`rbd snap rollback` modifies the **existing RBD image**.

This is different from creating a new volume from a snapshot.

``` text
Rollback:

Existing RBD
     │
     ▼
Old state
```

Whereas a clone/restore workflow can be:

``` text
Snapshot
    │
    ▼
New RBD
```

For Kubernetes-managed RBD volumes, prefer the Kubernetes CSI
`VolumeSnapshot` and restore workflow rather than manually rolling back
the underlying RBD without coordinating with Kubernetes.

## 10. Useful Commands

List RBD images:

``` bash
rbd ls -p rbd-pool-app1
```

Show image information:

``` bash
rbd info rbd-pool-app1/rbd-image-app1
```

List snapshots:

``` bash
rbd snap ls rbd-pool-app1/rbd-image-app1
```

Show mapped devices:

``` bash
rbd showmapped
```

List block devices:

``` bash
lsblk
```

Check filesystem:

``` bash
df -h
df -Th
```

## Key Takeaways

-   `rbd snap create` creates a point-in-time snapshot.
-   `rbd snap ls` lists snapshots.
-   Stop I/O and unmount before performing a rollback.
-   `rbd unmap` removes the RBD block-device mapping.
-   `rbd snap rollback` restores the existing RBD to the snapshot state.
-   After rollback, map and mount the RBD again.
-   RBD snapshots are not a substitute for independent backups.
-   For Kubernetes/CSI-managed volumes, use Kubernetes `VolumeSnapshot`
    and restore mechanisms where possible.
