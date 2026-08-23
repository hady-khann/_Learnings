# Ceph RBD: Resize an Image and Grow XFS

## Overview

Increasing an RBD volume requires two logical steps:

``` text
RBD Image
   │
   │ resize
   ▼
Larger Block Device
   │
   │ filesystem grow
   ▼
Larger XFS Filesystem
```

Increasing the RBD size alone does **not** automatically increase the
filesystem size.

## 1. Check the Current Size

Check the RBD:

``` bash
rbd info rbd-pool-app1/rbd-image-app1
```

Check the block device and filesystem:

``` bash
lsblk
df -h
```

Example starting point:

``` text
RBD        = 5G
/dev/rbd0  = 5G
XFS        = 5G
```

## 2. Increase the RBD Size

Resize the RBD from 5 GiB to 15 GiB:

``` bash
rbd resize \
  --image rbd-image-app1 \
  --size 15G \
  --pool rbd-pool-app1
```

Verify:

``` bash
rbd info rbd-pool-app1/rbd-image-app1
```

Check the block device:

``` bash
lsblk
```

The expected state is now:

``` text
RBD        = 15G
/dev/rbd0  = 15G
XFS        = 5G
```

The XFS filesystem has not grown yet.

## 3. Grow the XFS Filesystem

Because the filesystem is XFS:

``` bash
xfs_growfs -d /mnt/rbd-mount-app1
```

Then verify:

``` bash
df -h
```

Expected final state:

``` text
RBD        = 15G
/dev/rbd0  = 15G
XFS        = 15G
```

## 4. Complete Workflow

``` bash
rbd info rbd-pool-app1/rbd-image-app1

rbd resize \
  --image rbd-image-app1 \
  --size 15G \
  --pool rbd-pool-app1

lsblk

xfs_growfs -d /mnt/rbd-mount-app1

df -h
```

## 5. Important Concept

There are two separate layers:

``` text
        Ceph
          │
          ▼
      RBD = 15G
          │
          ▼
     /dev/rbd0
          │
          ▼
       XFS = 15G
```

The RBD resize changes the available block-device capacity.

`xfs_growfs` expands the filesystem to use that additional capacity.

## 6. EXT4

If the filesystem is EXT4 instead of XFS, the filesystem expansion
command is different:

``` bash
resize2fs /dev/rbd0
```

For XFS:

``` bash
xfs_growfs -d /mnt/rbd-mount-app1
```

## 7. Shrinking

Be careful with shrinking.

Increasing:

``` text
5G → 15G
```

is a normal operation.

Shrinking:

``` text
15G → 5G
```

is a different and much riskier operation.

XFS cannot be shrunk in place.

Therefore, do not simply run:

``` bash
rbd resize --size 5G ...
```

on an RBD containing a 15G XFS filesystem.

## Important Notes

-   Resize the RBD first.
-   Then grow the filesystem.
-   Always verify both layers with `rbd info`, `lsblk`, and `df -h`.
-   The filesystem must be able to grow into the new space.
