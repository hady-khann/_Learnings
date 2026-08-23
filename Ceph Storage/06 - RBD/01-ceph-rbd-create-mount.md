# Ceph RBD: Pool, Image, Map and Mount

## Overview

This document shows the basic workflow for creating a Ceph RBD image,
mapping it to a Linux block device, formatting it, and mounting it.

``` text
Ceph Pool
   │
   ▼
RBD Image
   │
   ▼
/dev/rbd0
   │
   ▼
XFS Filesystem
   │
   ▼
/mnt/rbd-mount-app1
```

## 1. Create the RBD Pool

Create a pool with 50 PGs:

``` bash
ceph osd pool create rbd-pool-app1 50
```

Set the replication factor to 3:

``` bash
ceph osd pool set rbd-pool-app1 size 3
```

Enable the RBD application on the pool:

``` bash
ceph osd pool application enable rbd-pool-app1 rbd
```

Verify:

``` bash
ceph osd pool get rbd-pool-app1 size
ceph osd pool application get rbd-pool-app1
```

> `size 3` means Ceph maintains three replicas of each replicated
> object. The actual placement is controlled by CRUSH.

## 2. Create the RBD Image

Create a 5 GiB RBD image:

``` bash
rbd create rbd-image-app1 \
  --pool rbd-pool-app1 \
  --size 5G
```

Verify:

``` bash
rbd info rbd-pool-app1/rbd-image-app1
```

## 3. Map the RBD Image

Map the image to a Linux block device:

``` bash
rbd map rbd-image-app1 --pool rbd-pool-app1
```

Check mapped RBD devices:

``` bash
rbd showmapped
```

The device may appear as:

``` text
/dev/rbd0
```

Verify the device:

``` bash
fdisk -l /dev/rbd0
```

You can also use:

``` bash
lsblk
```

## 4. Create a Filesystem

Create an XFS filesystem:

``` bash
mkfs.xfs /dev/rbd0
```

> `mkfs.xfs` destroys existing data on the device. Only run it on a new
> or intentionally empty RBD.

## 5. Mount the RBD

Create a mount point:

``` bash
mkdir -p /mnt/rbd-mount-app1
```

Mount the filesystem:

``` bash
mount /dev/rbd0 /mnt/rbd-mount-app1
```

Verify:

``` bash
df -h /mnt/rbd-mount-app1
df -Th
```

## Final State

``` text
rbd-pool-app1
      │
      └── rbd-image-app1 (5 GiB)
                │
                ▼
             /dev/rbd0
                │
                ▼
              XFS
                │
                ▼
       /mnt/rbd-mount-app1
```

## Important Notes

-   RBD is block storage; XFS is the filesystem placed on top of it.
-   `rbd map` exposes the RBD image as a Linux block device.
-   The pool replication setting and the filesystem are separate
    concepts.
-   Before using an RBD on another client, make sure the access pattern
    is safe. A normal XFS filesystem should not be mounted read/write
    simultaneously by multiple independent hosts.
