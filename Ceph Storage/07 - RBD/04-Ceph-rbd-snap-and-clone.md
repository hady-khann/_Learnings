<div dir="rtl" lang="fa">

# Ceph RBD — Image, Snapshot & Clone Workflow

مراحل ساخت یک RBD image پایه، گرفتن snapshot، محافظت از آن، ساخت clone از رویش، و unlink کردن clone با flatten.

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

خروجی باید clone را نشان دهد، مثلاً:

```text
rbd-pool-app1/rbd-image-app1-clone20260825
```

تا وقتی clone به snapshot والد وصل است، نمی‌توان snapshot را `unprotect` یا حذف کرد.

## ۸. جدا کردن clone از parent (unlink با flatten)

Clone به‌صورت Copy-on-Write به snapshot والد وابسته است. برای **unlink** کردن (مستقل کردن clone از parent)، داده‌های مشترک را داخل خود clone کپی کنید:

```bash
rbd flatten rbd-pool-app1/rbd-image-app1-clone20260825
```

بعد از flatten:

- clone دیگر parent ندارد و به‌تنهایی کامل است
- از لیست childrenِ snapshot حذف می‌شود
- snapshot را می‌توان `unprotect` کرد

## ۹. بررسی مجدد snapshot بعد از unlink

```bash
rbd info \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

لیست children باید خالی باشد:

```bash
rbd children \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

برای اطمینان از unlink شدن خود clone:

```bash
rbd info rbd-pool-app1/rbd-image-app1-clone20260825
```

در خروجی clone نباید فیلد `parent` دیده شود.

## ۱۰. برداشتن محافظت snapshot (unprotect)

وقتی هیچ child باقی نمانده باشد:

```bash
rbd snap unprotect \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

در صورت نیاز، حذف snapshot:

```bash
rbd snap rm \
  rbd-pool-app1/rbd-image-app1-base1@rbd-image-app1-snap20260825
```

---

## نکات مهم

- قبل از `rbd clone`، حتماً باید snapshot با `rbd snap protect` محافظت‌شده باشد؛ در غیر این صورت دستور clone با خطا مواجه می‌شود.
- مقصد در دستور `clone` هرگز نباید شامل `@` باشد — چون کلون یک **image جدید** می‌سازد، نه snapshot جدید.
- Unlink کلون از والد با `rbd flatten` انجام می‌شود؛ دستور جداگانه‌ای به نام `rbd unlink` وجود ندارد.
- `flatten` حجم و زمان می‌برد چون objectهای مشترک parent را داخل clone کپی می‌کند.
- برای حذف snapshot محافظت‌شده: اول flatten/حذف همه‌ی cloneها، بعد `rbd snap unprotect`، سپس `rbd snap rm`. با `rbd children` خالی بودن لیست را چک کنید.
</div>
