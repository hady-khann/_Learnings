# ساخت و Mount کردن RBD

این راهنما مراحل ایجاد یک Ceph RBD، ساخت Filesystem و Mount کردن آن روی سیستم را پوشش می‌دهد.

## 1. ساخت Pool

```bash
ceph osd pool create rbd-pool-app1 50
```

## 2. تعیین Replica = 3

```bash
ceph osd pool set rbd-pool-app1 size 3
ceph osd pool application enable rbd-pool-app1 rbd
```

با این تنظیم، هر داده در Pool دارای **۳ Replica** خواهد بود.

## 3. ساخت RBD Image با حجم 5GB

```bash
rbd create rbd-image-app1 \
  --pool rbd-pool-app1 \
  --size 5G
```

## 4. بررسی Image

```bash
rbd info rbd-pool-app1/rbd-image-app1
```

## 5. Map کردن RBD به Block Device

```bash
rbd map rbd-image-app1 --pool rbd-pool-app1
```

پس از موفقیت، RBD به یک Block Device مانند `/dev/rbd0` متصل می‌شود.

## 6. مشاهده RBDهای Map شده

```bash
rbd showmapped
```

## 7. بررسی Block Device

```bash
fdisk -l /dev/rbd0
```

## 8. ساخت Filesystem

در این مثال از XFS استفاده می‌کنیم:

```bash
mkfs.xfs /dev/rbd0
```

> **توجه:** اجرای `mkfs` روی Device موجود، اطلاعات قبلی آن را از بین می‌برد. این دستور را فقط روی RBD جدید اجرا کنید.

## 9. ساخت Mount Point

```bash
mkdir -p /mnt/rbd-mount-app1
```

## 10. Mount کردن RBD

```bash
mount /dev/rbd0 /mnt/rbd-mount-app1
```

## 11. بررسی

```bash
df -h /mnt/rbd-mount-app1
df -Th
```

در صورت موفقیت، باید `/dev/rbd0` را به همراه Filesystem نوع `xfs` و Mount Point زیر مشاهده کنید:

```text
/mnt/rbd-mount-app1
```
