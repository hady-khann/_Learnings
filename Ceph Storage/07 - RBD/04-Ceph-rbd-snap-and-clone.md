# Ceph RBD — Image, Snapshot & Clone Workflow

مراحل ساخت یک RBD image پایه، گرفتن snapshot، محافظت از آن، و ساخت clone از رویش.

## ۱. ساخت image پایه

```bash
rbd create rbd-image-app1-base1 \
  --pool rbd-pool-app1 \
  --image-feature layering \
  --size 5G
```

## ۲. بررسی اطلاعات image

```bash
rbd info rbd-pool-app1/rbd-image-app1-base1
```

## ۳. ساخت snapshot از image پایه

```bash
rbd snap create \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

## ۴. محافظت از snapshot (پیش‌نیاز کلون گرفتن)

```bash
rbd snap protect \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

## ۵. ساخت clone از روی snapshot

```bash
rbd clone \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825 \
  rbd-pool-app1/rbd-image-app1-clone20260825 \
  --image-feature layering
```

## ۶. بررسی اطلاعات snapshot

```bash
rbd info \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

## ۷. دیدن لیست clone‌های ساخته‌شده از این snapshot

```bash
rbd children \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

---

## نکات مهم

- قبل از `rbd clone`، حتماً باید snapshot با `rbd snap protect` محافظت‌شده باشد؛ در غیر این صورت دستور clone با خطا مواجه می‌شود.
- مقصد در دستور `clone` هرگز نباید شامل `@` باشد — چون کلون یک **image جدید** می‌سازد، نه snapshot جدید.
- برای حذف snapshot محافظت‌شده، ابتدا باید با `rbd snap unprotect` محافظتش را بردارید و مطمئن شوید هیچ clone‌ای از رویش باقی نمانده (با `rbd children` چک کنید).
