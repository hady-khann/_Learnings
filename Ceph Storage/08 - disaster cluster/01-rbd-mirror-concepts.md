<div dir="rtl" lang="fa">

# Disaster Cluster و RBD Mirroring

جلسهٔ آنلاین پنجم (Iran Linux House) دربارهٔ بازیابی از فاجعه با **RBD mirroring** بین دو کلاستر Ceph است: یک سایت Active و یک سایت Passive.

نام‌ها در یادداشت با سبک `07 - RBD` هستند (`rbd-pool-app1`، `rbd-image-appN`). معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| `data` | `rbd-pool-app1` |
| `image-1` … `image-4` | `rbd-image-app1` … `rbd-image-app4` |
| `anisa-1` / `anisa-2` | `rbd-image-app5` / `rbd-image-app6` |

## ایدهٔ کلی

اول فرق دو لایهٔ کپی را جدا کنید؛ وگرنه «replica» در Pool با «mirror» بین سایت‌ها قاطی می‌شود.

| لایه | کجا | چه محافظت می‌کند | نمونه |
|---|---|---|---|
| Replica داخل کلاستر (`size=3` در پوشهٔ `06 - Pool`) | همان FSID، همان سه نود | خرابی دیسک یا یک OSD | سه کپی روی OSDهای سایت Active |
| RBD mirror بین کلاسترها | دو FSID جدا، دو سایت | خرابی کل سایت (آتش، برق، شبکهٔ دیتاسنتر) | کپی Image روی کلاستر `backup` |

اضافه کردن OSD به **همان** کلاستر، فاجعهٔ سایت را نجات نمی‌دهد: اگر رک Active از بین برود، هر سه replica هم با آن می‌روند. برای Disaster Recovery باید کلاستر دومی با MON/OSD مال خودش داشته باشید.

RBD mirroring یک **replication ناهمگام (asynchronous)** از Imageهای RBD بین چند کلاستر Ceph است. ناهمگام یعنی سایت Passive چند لحظه عقب‌تر است؛ نوشتن روی Active منتظر تأیید سایت مقابل نمی‌ماند.

یک replica با نقطهٔ زمانی مشخص (point-in-time) از هر تغییر روی Image ساخته می‌شود؛ از جمله:

- snapshot
- clone
- IOPS خواندن و نوشتن
- تغییر اندازهٔ Block Device

نتیجه: یک کپی **crash-consistent** از Image سمت مقابل، روی سایت محلی در دسترس است. crash-consistent یعنی اگر Active همین حالا خاموش شود، کپی Passive مثل دیسکی است که برق کشیده شده — فایل‌سیستم معمولاً با journal خودش درست می‌شود، ولی تراکنش نیمه‌کارهٔ دیتابیس ممکن است commit نشده باشد. این با application-consistent فرق دارد (آنجا اول خود اپ snapshot می‌گیرد).

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
- هر دو سمت daemon مربوط به `rbd-mirror` دارند؛ جهت همگام‌سازی با **peer** مشخص می‌شود (`rx-tx`). Peer یعنی ثبت کلاستر مقابل؛ بدون آن mirroring مبدأ ندارد. شرح کامل در `03-rbd-mirror-lab.md`.

Mirroring می‌تواند **active+passive** یا **active+active** باشد. در این لاب مدل Active/Passive پیاده شد.

## پیش‌نیاز Image: journaling و exclusive-lock

قبل از روشن کردن mirroring روی کلاستر، Featureهای زیر باید روی Image فعال باشند:

```text
exclusive-lock
journaling
```

**journaling چیست؟** یک write-ahead log روی خود Image: هر نوشتن اول به ترتیب در journal ثبت می‌شود، بعد روی داده. daemon سمت Passive همین journal را می‌خواند و **replay** می‌کند (دوباره همان ترتیب را روی کپی محلی اجرا می‌کند). بدون journal، سایت مقابل نمی‌داند کدام بلاک‌ها به چه ترتیبی عوض شده‌اند.

**exclusive-lock چیست؟** قفلی که در هر لحظه فقط یک کلاینت «مالک نوشتن» Image باشد. برای mirror لازم است چون فقط **یک طرف primary** اجازهٔ نوشتن دارد؛ وگرنه دو سایت همزمان می‌نویسند و journal قابل replay نیست.

بدون این دو Feature، mirroring کار نمی‌کند.

نمونهٔ ساخت Image مناسب برای mirror:

```bash
rbd create rbd-image-app6 \
  --size 1024 \
  --pool rbd-pool-app1 \
  --image-feature exclusive-lock,journaling
```

## دو حالت Pool

| Mode    | معنی | کی استفاده کنید |
| ------- | ---- | --- |
| `pool`  | همهٔ Imageهای Pool (با journaling) به‌صورت خودکار mirror می‌شوند | وقتی کل Pool اپ باید در سایت DR باشد — همین لاب |
| `image` | فقط Imageهایی که جداگانه enable شده‌اند | وقتی چند Image در Pool هست و فقط بعضی باید replicate شوند |

`rbd mirror pool enable … pool` فقط می‌گوید «این Pool *اجازه* دارد mirror شود». هنوز سایت مقابل را نمی‌شناسد؛ آن کار با **peer** است (فایل `03-rbd-mirror-lab.md`).

در این لاب از **pool mode** استفاده شد:

```bash
rbd mirror pool enable rbd-pool-app1 pool
```

## نام کلاستر در این لاب

| نقش | نودها | نام Cluster |
| --- | ----- | ----------- |
| Active / primary | `ceph-node1`, `ceph-node2`, `ceph-node3` | `ceph` (پیش‌فرض) |
| Passive / disaster | `ceph-node5`, `ceph-node6`, `ceph-node7` | `backup` |
| Pool | هر دو سمت | `rbd-pool-app1` |

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
</div>
