# Disaster Cluster و RBD Mirroring

جلسهٔ آنلاین پنجم (Iran Linux House) دربارهٔ بازیابی از فاجعه با **RBD mirroring** بین دو کلاستر Ceph است: یک سایت Active و یک سایت Passive.

## ایدهٔ کلی

RBD mirroring یک **replication ناهمگام (asynchronous)** از Imageهای RBD بین چند کلاستر Ceph است.

یک replica با نقطهٔ زمانی مشخص (point-in-time) از هر تغییر روی Image ساخته می‌شود؛ از جمله:

- snapshot
- clone
- IOPS خواندن و نوشتن
- تغییر اندازهٔ Block Device

نتیجه: یک کپی **crash-consistent** از Image سمت مقابل، روی سایت محلی در دسترس است.

## معماری Active / Passive

```text
ACTIVE CLUSTER                      PASSIVE CLUSTER
┌──────────────┐   SYNC             ┌──────────────┐
│ MON MON MON  │ ◄──────────────►   │ RBD MIRROR   │
│ RBD MIRROR   │                    │ MON MON MON  │
│ OSD × N      │                    │ OSD × N      │
└──────────────┘                    └──────────────┘
```

- کلاستر Active داده را می‌نویسد.
- کلاستر Passive با `rbd-mirror` همان Imageها را replay می‌کند.
- هر دو سمت daemon مربوط به `rbd-mirror` دارند؛ جهت همگام‌سازی با peer مشخص می‌شود (`rx-tx`).

Mirroring می‌تواند **active+passive** یا **active+active** باشد. در این لاب مدل Active/Passive پیاده شد.

## پیش‌نیاز Image: journaling و exclusive-lock

قبل از روشن کردن mirroring روی کلاستر، Featureهای زیر باید روی Image فعال باشند:

```text
exclusive-lock
journaling
```

Journaling همهٔ تغییرات Image را **به همان ترتیبی که رخ داده‌اند** ثبت می‌کند. بدون این دو Feature، mirroring کار نمی‌کند.

نمونهٔ ساخت Image مناسب برای mirror:

```bash
rbd create anisa-2 \
  --size 1024 \
  --pool data \
  --image-feature exclusive-lock,journaling
```

## دو حالت Pool

| Mode    | معنی |
| ------- | ---- |
| `pool`  | همهٔ Imageهای Pool (با journaling) به‌صورت خودکار mirror می‌شوند |
| `image` | فقط Imageهایی که جداگانه enable شده‌اند |

در این لاب از **pool mode** استفاده شد:

```bash
rbd mirror pool enable data pool
```

## نام کلاستر در این لاب

| نقش | نودها | نام Cluster |
| --- | ----- | ----------- |
| Active / primary | `ceph-node1`, `ceph-node2`, `ceph-node3` | `ceph` (پیش‌فرض) |
| Passive / disaster | `ceph-node5`, `ceph-node6`, `ceph-node7` | `backup` |
| Pool | هر دو سمت | `data` |

نسخهٔ پکیج `rbd-mirror` در ویدیو:

```text
15.2.17  (Octopus)
```

اگر نام Cluster پیش‌فرض `ceph` نباشد، دستورها باید `--cluster <name>` بگیرند، یا:

```bash
export CEPH_ARGS="--cluster backup"
```

بعد از این export، دستورهایی مثل `ceph -s` روی کلاستر disaster کار می‌کنند.

> **هشدار:** اگر `CEPH_ARGS` یا `--cluster` اشتباه باشد، خطای زیر دیده می‌شود: `Error initializing cluster client: ObjectNotFound('RADOS object not found (error read_file)')`.
