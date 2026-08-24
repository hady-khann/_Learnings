# افزایش حجم RBD و Filesystem

این راهنما افزایش حجم یک RBD از **5GB به 15GB** و سپس افزایش Filesystem مربوطه را نشان می‌دهد.

## 1. افزایش حجم RBD

```bash
rbd resize \
  --image rbd-image-app1 \
  --size 15G \
  --pool rbd-pool-app1
```

در این مرحله حجم Virtual Block Device در Ceph افزایش پیدا می‌کند.

## 2. بررسی حجم

```bash
df -h
```

> در این مرحله ممکن است Filesystem هنوز حجم قبلی را نشان دهد. افزایش حجم RBD به‌تنهایی باعث افزایش ظرفیت Filesystem نمی‌شود.

## 3. افزایش حجم XFS

با توجه به اینکه Filesystem این RBD از نوع XFS است:

```bash
xfs_growfs -d /mnt/rbd-mount-app1
```

## 4. بررسی نهایی

```bash
df -h
```

اکنون Filesystem باید فضای افزایش‌یافته را در اختیار داشته باشد و حجم آن تقریباً **15GB** باشد.
