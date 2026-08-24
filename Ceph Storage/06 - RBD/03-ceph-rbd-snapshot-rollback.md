# ایجاد Snapshot و Rollback کردن RBD

این راهنما نحوه ایجاد Snapshot از RBD، بررسی Snapshot، Unmap کردن RBD، انجام Rollback و سپس Map و Mount مجدد آن را نشان می‌دهد.

## 1. ایجاد Snapshot

```bash
rbd snap create rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823
```

Snapshot با نام زیر ایجاد می‌شود:

```text
rbd-image-app1-20260823
```

## 2. مشاهده Snapshotها

```bash
rbd snap ls rbd-pool-app1/rbd-image-app1
```

## 3. Unmount کردن Filesystem

قبل از Rollback، Filesystem را Unmount کنید:

```bash
umount /mnt/rbd-mount-app1
```

## 4. Unmap کردن RBD

```bash
rbd unmap /dev/rbd0
```

برای اطمینان از Unmap شدن:

```bash
rbd showmapped
```

## 5. Rollback به Snapshot

```bash
rbd snap rollback \
  rbd-pool-app1/rbd-image-app1@rbd-image-app1-20260823
```

این عملیات وضعیت RBD را به زمان ایجاد Snapshot برمی‌گرداند.

> **هشدار:** Rollback یک عملیات مخرب است و تغییرات RBD از زمان ایجاد Snapshot تا زمان Rollback را کنار می‌گذارد. قبل از اجرای آن، مطمئن شوید Snapshot موردنظر همان نقطه بازیابی مطلوب است.

## 6. Map کردن مجدد RBD

```bash
rbd map rbd-pool-app1/rbd-image-app1
```

بررسی کنید Device صحیح ایجاد شده باشد:

```bash
rbd showmapped
```

## 7. Mount کردن مجدد

```bash
mount /dev/rbd0 /mnt/rbd-mount-app1
```

## 8. بررسی نهایی

```bash
df -h /mnt/rbd-mount-app1
```

در پایان، RBD باید دوباره روی Mount Point زیر در دسترس باشد:

```text
/mnt/rbd-mount-app1
```
