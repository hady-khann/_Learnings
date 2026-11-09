# مستند راه‌اندازی Mirroring برای CephFS

## ۱. مرور کلی

- مکانیزم: daemon به نام `cephfs-mirror` روی کلاستر **مبدأ** اجرا می‌شود و **snapshot های** دایرکتوری‌های انتخاب‌شده را به کلاستر **مقصد** sync می‌کند.
- جهت: یک‌طرفه (مبدأ ← مقصد).
- نوع: ناهمزمان و مبتنی بر snapshot. تغییراتی که snapshot نشده باشند منتقل نمی‌شوند.
- هدف: Disaster Recovery، نه جایگزین backup (حذف یا خرابی داده هم در snapshot بعدی به مقصد می‌رسد).

### فرضیات این مستند

| مورد | مقدار |
|---|---|
| مبدأ | کلاستر `site-a` |
| مقصد | کلاستر `site-b` |
| نام filesystem | `cephfs` |
| دایرکتوری mirror شونده | `/data` |
| روش نصب | cephadm (Quincy یا جدیدتر) |

> دستورها بر اساس مستندات رسمی Ceph نوشته شده‌اند. قبل از اجرا روی محیط واقعی، روی لب تست کنید و با `ceph versions` نسخه را بررسی کنید، چون syntax بعضی دستورها بین نسخه‌ها کمی فرق می‌کند.

---

## ۲. پیش‌نیازها

روی **هر دو** کلاستر:

```bash
ceph -s                 # باید HEALTH_OK باشد
ceph versions           # ترجیحاً هر دو کلاستر هم‌نسخه (major) باشند
ceph fs ls              # filesystem باید وجود داشته باشد
```

ملاحظات شبکه:

- نودی که `cephfs-mirror` روی آن اجرا می‌شود باید به mon ها (پورت‌های 3300 و 6789)، OSD ها (بازه 6800 تا 7300) و MDS های کلاستر مقصد دسترسی داشته باشد.
- ساعت سرورها با chrony یا NTP هماهنگ باشد.
- اگر لب روی Vagrant است، مطمئن شوید دو کلاستر روی یک شبکه مشترک (private network) هستند. این رایج‌ترین دلیل شکست peering است.

تست اتصال از نود mirror به مقصد:

```bash
nc -zv <site-b-mon-ip> 3300
nc -zv <site-b-mon-ip> 6789
```

در صورت نیاز filesystem مقصد را بسازید:

```bash
# روی site-b
ceph fs volume create cephfs
```

---

## ۳. مراحل اولیه cephx

cephx سیستم احراز هویت Ceph است. هر کاربر یک **entity** با **key** و **caps** (مجوزها) دارد.

دستورهای پایه:

```bash
ceph auth ls                       # فهرست همه کاربران
ceph auth get client.admin         # دیدن key و caps یک کاربر
ceph auth get-or-create client.x mon 'allow r' osd 'allow rw'   # ساخت کاربر
ceph auth rm client.x              # حذف
```

در CephFS mirroring دو نوع کاربر درگیر است:

1. **کاربر daemon روی مبدأ** (مثل `client.cephfs-mirror.<host>.<id>`): cephadm خودش می‌سازد و کار خاصی لازم نیست.
2. **کاربر ریموت روی مقصد**: daemon مبدأ با این کاربر به کلاستر مقصد وصل می‌شود. باید ساخته شود.

### ساخت کاربر ریموت روی مقصد (site-b)

```bash
ceph fs authorize cephfs client.mirror_remote / rwps
```

- `/` یعنی دسترسی روی کل filesystem.
- `rwps` یعنی خواندن، نوشتن، ساخت snapshot و تغییر quota. این سطح دسترسی برای mirror لازم است.
- کپی‌کردن key لازم نیست؛ در مرحله bootstrap داخل token قرار می‌گیرد.

بررسی:

```bash
ceph auth get client.mirror_remote
```

---

## ۴. مراحل راه‌اندازی

### مرحله ۱: فعال‌سازی ماژول mirroring (روی هر دو کلاستر)

```bash
ceph mgr module enable mirroring
```

### مرحله ۲: اجرای daemon روی مبدأ (site-a)

```bash
ceph orch apply cephfs-mirror
# یا با مشخص‌کردن نود:
ceph orch apply cephfs-mirror --placement="<hostname>"

ceph orch ps | grep cephfs-mirror     # وضعیت باید running باشد
```

> اگر cephadm ندارید: بسته `cephfs-mirror` را نصب کنید و سرویس systemd آن را با یک کاربر cephx مناسب اجرا کنید.

### مرحله ۳: فعال‌سازی mirroring برای filesystem (site-a)

```bash
ceph fs snapshot mirror enable cephfs
```

### مرحله ۴: ساخت bootstrap token روی مقصد (site-b)

```bash
ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b
```

- آرگومان‌ها به ترتیب: نام filesystem، entity ساخته‌شده در بخش cephx، و یک نام دلخواه برای شناسایی سایت.
- خروجی یک JSON با فیلد `token` است. آن را کپی کنید (مثل یک رمز با آن رفتار کنید، چون دسترسی به کلاستر مقصد را می‌دهد).

### مرحله ۵: import کردن token روی مبدأ (site-a)

```bash
ceph fs snapshot mirror peer_bootstrap import cephfs <TOKEN>
```

### مرحله ۶: افزودن دایرکتوری برای mirror (site-a)

ابتدا دایرکتوری را داخل filesystem بسازید (mount کنید):

```bash
mkdir -p /mnt/cephfs/data
```

سپس:

```bash
ceph fs snapshot mirror add cephfs /data
```

---

## ۵. ساخت snapshot

هیچ داده‌ای بدون snapshot منتقل نمی‌شود.

### دستی (برای تست)

```bash
echo "test" > /mnt/cephfs/data/file1.txt
mkdir /mnt/cephfs/data/.snap/snap1
```

اگر خطای مجوز snapshot گرفتید:

```bash
ceph fs set cephfs allow_new_snaps true
```

### خودکار (برای استفاده دائمی)

```bash
ceph mgr module enable snap_schedule
ceph fs snap-schedule add /data 1h
ceph fs snap-schedule retention add /data h 24     # نگه‌داشتن ۲۴ snapshot ساعتی
ceph fs snap-schedule status /data
```

---

## ۶. بررسی و تست

روی مبدأ:

```bash
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror daemon status
```

روی مقصد، mount کنید و snapshot را ببینید:

```bash
mount -t ceph :/ /mnt/cephfs-dr -o name=admin,fs=cephfs
ls /mnt/cephfs-dr/data/.snap/
ls /mnt/cephfs-dr/data/
```

بعد از چند ثانیه تا چند دقیقه (بسته به حجم)، snapshot و محتوای آن باید روی مقصد دیده شود.

---

## ۷. عیب‌یابی

| علامت | بررسی |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps`، `ceph log last cephadm`، لاگ کانتینر با `cephadm logs --name cephfs-mirror.<id>` |
| peer اضافه نمی‌شود | اتصال شبکه به mon های مقصد، اعتبار token، نام filesystem |
| داده منتقل نمی‌شود | آیا snapshot ساخته شده؟ آیا مسیر با `snapshot mirror add` اضافه شده؟ |
| failed در status | `ceph fs snapshot mirror daemon status` و لاگ daemon |
| authentication error | caps کاربر `client.mirror_remote` در مقصد را با `ceph auth get` بررسی کنید |

---

## ۸. حذف و غیرفعال‌سازی

```bash
ceph fs snapshot mirror remove cephfs /data          # حذف یک دایرکتوری
ceph fs snapshot mirror peer_list cephfs             # گرفتن uuid مربوط به peer
ceph fs snapshot mirror peer_remove cephfs <uuid>    # حذف peer
ceph fs snapshot mirror disable cephfs               # غیرفعال‌کردن کامل
```

---

## ۹. نکات مهم

- روی دایرکتوری‌هایی که mirror می‌شوند در مقصد **ننویسید**؛ تغییرات شما در sync بعدی بازنویسی می‌شود.
- این mirroring یک‌طرفه است. برای failover باید مقصد را به‌صورت دستی به‌عنوان filesystem فعال استفاده کنید و برگشت (failback) فرآیند جدایی دارد.
- حجم زیاد داده و تعداد زیاد فایل‌های کوچک، sync را کند می‌کند.
- برای backup واقعی (برگشت به نقطه زمانی)، snapshot های retention دار و در صورت نیاز ابزار backup جدا هم نگه دارید.
