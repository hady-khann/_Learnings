# مستند راه‌اندازی Mirroring برای RBD

## ۱. مرور کلی

- مکانیزم: daemon به نام `rbd-mirror` تغییرات image ها را بین دو کلاستر replicate می‌کند.
- ناهمزمان است و برای Disaster Recovery طراحی شده، نه backup. حذف یا خرابی داده هم به کلاستر دوم می‌رسد.
- دو حالت mirroring:

| حالت | روش کار | مزیت | عیب |
|---|---|---|---|
| **journal-based** | هر نوشتن در journal ثبت و روی مقصد replay می‌شود | RPO نزدیک به صفر | سربار نوشتن روی دیسک، نیاز به feature `journaling` |
| **snapshot-based** | snapshot های دوره‌ای گرفته می‌شود و تفاوت‌ها منتقل می‌شود | سبک‌تر | RPO برابر با بازه snapshot |

- دو سطح فعال‌سازی:
  - **image mode**: انتخاب تک‌تک image ها (توصیه‌شده، انعطاف‌پذیر).
  - **pool mode**: همه image های دارای journaling در pool.
- جهت: یک‌طرفه (`rx-only` در مقصد) یا دوطرفه (`rx-tx`، لازم برای failback).

### فرضیات

| مورد | مقدار |
|---|---|
| مبدأ | `site-a` |
| مقصد | `site-b` |
| pool | `rbd` |
| image نمونه | `img1` |
| روش نصب | cephadm (Quincy یا جدیدتر) |

> دستورها بر اساس مستندات رسمی Ceph است. قبل از production روی لب تست کنید و با `ceph versions` نسخه را بررسی کنید.

---

## ۲. پیش‌نیازها

روی هر دو کلاستر:

```bash
ceph -s              # HEALTH_OK
ceph versions
```

- نودی که `rbd-mirror` اجرا می‌شود به mon ها (3300 و 6789) و OSD ها (6800 تا 7300) کلاستر طرف مقابل دسترسی شبکه داشته باشد.
- ساعت‌ها با NTP/chrony هماهنگ باشد.
- pool با **همان نام** روی هر دو کلاستر وجود داشته باشد.

---

## ۳. مراحل اولیه cephx

```bash
ceph auth ls
ceph auth get client.admin
ceph auth get-or-create client.x mon 'allow r' osd 'allow rw'
```

در RBD mirroring دو کاربر مهم هست:

1. **کاربر daemon** (`client.rbd-mirror.<id>`): با caps مخصوص `profile rbd-mirror` به کلاستر محلی وصل می‌شود. cephadm خودش می‌سازد.
2. **کاربر peer** (`client.rbd-mirror-peer`): هنگام ساخت bootstrap token به‌صورت خودکار روی کلاستر مبدأ ساخته می‌شود و caps آن `profile rbd-mirror-peer` است. daemon طرف مقابل با آن به این کلاستر وصل می‌شود.

اگر **بدون cephadm** کار می‌کنید، کاربر daemon را دستی بسازید:

```bash
ceph auth get-or-create client.rbd-mirror.site-b \
  mon 'profile rbd-mirror' osd 'profile rbd' \
  -o /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
```

بررسی:

```bash
ceph auth get client.rbd-mirror.site-b
```

---

## ۴. مراحل راه‌اندازی (یک‌طرفه: site-a به site-b)

دستورها را روی نود ادمین هر کلاستر اجرا کنید (با cephadm: `cephadm shell`).

### مرحله ۱: ساخت و آماده‌سازی pool (روی هر دو کلاستر)

```bash
ceph osd pool create rbd
rbd pool init rbd
```

### مرحله ۲: فعال‌سازی mirroring روی pool (روی هر دو کلاستر، با یک mode)

```bash
rbd mirror pool enable rbd image
```

> اگر pool mode می‌خواهید به‌جای `image` بنویسید `pool`.

### مرحله ۳: ساخت bootstrap token (روی site-a)

```bash
rbd mirror pool peer bootstrap create --site-name site-a rbd > /root/rbd-token
```

فایل token را امن به site-b منتقل کنید (مثلاً با scp). مثل رمز با آن رفتار کنید.

### مرحله ۴: import کردن token (روی site-b)

برای حالت **یک‌طرفه**:

```bash
rbd mirror pool peer bootstrap import --site-name site-b --direction rx-only rbd /root/rbd-token
```

برای **دوطرفه** (لازم برای failback) گزینه `--direction rx-only` را حذف کنید.

### مرحله ۵: اجرای daemon

در حالت یک‌طرفه فقط روی **site-b** (دریافت‌کننده)، و در حالت دوطرفه روی هر دو:

```bash
# با cephadm
ceph orch apply rbd-mirror --placement=1
ceph orch ps | grep rbd-mirror
```

بدون cephadm (بسته apt):

```bash
apt install rbd-mirror
systemctl enable --now ceph-rbd-mirror@rbd-mirror.site-b
```

(کلید کاربر باید در `/etc/ceph/ceph.client.rbd-mirror.site-b.keyring` باشد.)

### مرحله ۶: ساخت image و فعال‌سازی mirroring روی آن (site-a)

**حالت journal-based:**

```bash
rbd create rbd/img1 --size 1G --image-feature exclusive-lock,journaling
rbd mirror image enable rbd/img1 journal
```

برای image موجود:

```bash
rbd feature enable rbd/img1 exclusive-lock,journaling
rbd mirror image enable rbd/img1 journal
```

**حالت snapshot-based:**

```bash
rbd create rbd/img1 --size 1G
rbd mirror image enable rbd/img1 snapshot

# تنظیم زمان‌بندی (هر ۵ دقیقه، برای کل pool)
rbd mirror snapshot schedule add --pool rbd 5m
rbd mirror snapshot schedule ls --pool rbd -R

# گرفتن دستی یک snapshot mirror
rbd mirror image snapshot rbd/img1
```

---

## ۵. بررسی و تست

روی هر دو کلاستر:

```bash
rbd mirror pool info rbd
rbd mirror pool status rbd --verbose
rbd mirror image status rbd/img1
```

وضعیت‌های مهم:

| وضعیت | معنی |
|---|---|
| `up+replaying` | روی مقصد در حال replicate است (سالم) |
| `up+syncing` | همگام‌سازی اولیه در جریان است |
| `up+stopped` | روی طرف primary، طبیعی |
| `down+unknown` | daemon اجرا نیست یا ارتباط قطع است |
| `up+error` | خطا، جزئیات در `description` |

تست عملی: روی site-a داده بنویسید (image را map و mount کنید)، روی site-b با `rbd mirror image status` ببینید replicate می‌شود. برای دیدن محتوا باید image را promote کنید (فقط در تست failover).

---

## ۶. Failover و Failback

### Failover برنامه‌ریزی‌شده (نیازمند mirroring دوطرفه)

```bash
# روی site-a
rbd mirror image demote rbd/img1
# روی site-b
rbd mirror image promote rbd/img1
```

### Failover اضطراری (site-a از دسترس خارج شده)

```bash
# روی site-b
rbd mirror image promote --force rbd/img1
```

### Failback بعد از برگشت site-a

```bash
# روی site-a
rbd mirror image demote rbd/img1
rbd mirror image resync rbd/img1
# بعد از همگام‌شدن، در صورت نیاز جهت‌ها را برگردانید
```

---

## ۷. عیب‌یابی

| علامت | بررسی |
|---|---|
| daemon نیست | `ceph orch ps`، `cephadm logs --name rbd-mirror.<id>` |
| `down+unknown` | شبکه، اجرای daemon، اعتبار کاربر peer |
| `up+error` | `rbd mirror image status rbd/img1` و بخش description |
| split-brain بعد از force promote | resync با `rbd mirror image resync` روی طرف قدیمی |
| replay کند | حجم نوشتن، سرعت شبکه، منابع نود mirror |

---

## ۸. حذف و غیرفعال‌سازی

```bash
rbd mirror image disable rbd/img1
rbd mirror pool disable rbd
rbd mirror pool peer remove rbd <peer-uuid>
```

---

## ۹. نکات مهم

- Mirroring برابر با backup نیست. برای برگشت به نقطه زمانی از snapshot های دوره‌ای یا `rbd export-diff` استفاده کنید.
- در حالت یک‌طرفه، failback خودکار وجود ندارد.
- حالت journal برای نوشتن سنگین، سربار قابل‌توجه دارد؛ در صورت نیاز به RPO نه‌چندان کم، snapshot-based ساده‌تر است.
- همیشه بعد از هر تغییر، `rbd mirror pool status --verbose` را بررسی کنید.
