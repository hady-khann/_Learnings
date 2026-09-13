<div dir="rtl" lang="fa">

# CephFS: فایل‌سیستم توزیع‌شده روی کلاستر

جلسهٔ آنلاین ششم (Iran Linux House، ۱۵ نوامبر ۲۰۲۲) با **CephFS** شروع می‌شود: یک POSIX filesystem روی همان RADOS که RBD از آن استفاده می‌کند.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| pool `anisa` | `rbd-pool-app1` |
| `/mnt/anisa-1` | `/mnt/cephfs-mount-app1` |

برخلاف RBD که یک Block Device به کلاینت می‌دهد، CephFS یک درخت فایل مشترک (`/` تا فایل‌ها) روی چند کلاینت mount می‌شود.

## اجزا

CephFS به سه چیز نیاز دارد:

| جزء | نقش |
|---|---|
| **MDS** (`ceph-mds`) | Metadata: نام فایل، دایرکتوری، permission، inode |
| Pool **`cephfs_metadata`** | محل ذخیرهٔ metadata |
| Pool **`cephfs_data`** | محل ذخیرهٔ محتوای فایل‌ها (objectهای RADOS) |

دادهٔ فایل مستقیم بین کلاینت و OSD رد و بدل می‌شود. MDS مسیر را باز می‌کند و بعد کنار می‌رود؛ bottleneck اصلی معمولاً MDS است نه OSD.

در این لاب MDS روی `ceph-node2` بود:

```text
cephfs:1 {0=ceph-node2=up:active}
```

عدد `0` یعنی rank صفر (MDS فعال). اگر بعد از ریبوت `up:active(laggy or crashed)` دیدید، mount کلاینت با خطای `no mds server is up or the cluster is laggy` شکست می‌خورد تا MDS دوباره active شود.

## راه‌های دسترسی کلاینت

اسلاید جلسه این مسیرها را نشان می‌دهد:

```text
CLIENTS
  YOUR APP ──► Ceph Cluster          (مستقیم / kernel / fuse)
  GANESHA   ──► NFS
  SAMBA     ──► CIFS
  HADOOP    ──► SMB
  KERNEL / Ceph-fuse
```

- **Kernel mount** (`mount -t ceph`) — در لاب همین استفاده شد؛ کرنل کلاینت `5.4.0-131-generic` بود.
- **ceph-fuse** — userspace؛ وقتی ماژول کرنل در دسترس نیست.
- **NFS-Ganesha / Samba** — در این جلسه فقط اشاره شد («خیلی کوتاه»)، لاب جدا اجرا نشد.
- **ceph-dokan** — در چت کلاس برای mount روی Windows پیشنهاد شد.

## نام Filesystem در برابر نام کاربر CephX

خروجی `ceph fs ls` در لاب:

```text
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

- `cephfs` = نام Filesystem
- `client.fs` = کاربر CephX که برای mount ساخته شد

گزینهٔ `name=` در دستور `mount` باید **شناسهٔ کاربر** باشد (`fs`)، نه نام Filesystem. اگر `name=cephfs` بدهید، کلاستر کاربر `client.cephfs` را می‌خواهد و معمولاً `Permission denied` می‌گیرید.

## وضعیت Poolها قبل از mount

روی `ceph-node1`:

```bash
ceph mds stat
ceph osd pool ls
ceph fs ls
ceph df
```

Poolهای دیده‌شده:

```text
device_health_metrics
anis
rbd-pool-app1
rbd
cephfs_data
cephfs_metadata
```

RAW STORAGE حدود `180 GiB` با ۹ OSD. خود Filesystem از قبل روی کلاستر ساخته شده بود؛ کار جلسه ساخت کاربر و mount روی `client-node1` بود.
</div>
