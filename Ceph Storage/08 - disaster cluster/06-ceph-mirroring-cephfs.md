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

## ۳. مراحل اولیه cephx (احراز هویت)

### ۳.۱ مفاهیم پایه

- **entity**: نام هر کاربر یا daemon در Ceph، مثل `client.admin`، `client.rbd-mirror.site-b` یا `osd.0`.
- **key**: رمز (secret) آن entity.
- **caps**: مجوزها به تفکیک سرویس‌ها (`mon`، `osd`، `mds`، `mgr`). هر کاربر فقط کاری را می‌تواند بکند که caps اجازه می‌دهد.
- **keyring**: فایلی که entity و key را نگه می‌دارد. شکل آن:

```
[client.rbd-mirror.site-b]
        key = AQxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
```

- **مهم: کاربران در دیتابیس mon های هر کلاستر ذخیره می‌شوند و هر کلاستر دیتابیس کاربران جدا دارد.** کاربری که روی `site-a` ساخته‌اید روی `site-b` وجود ندارد، و برعکس. به همین دلیل باید بدانید هر دستور را روی کدام کلاستر می‌زنید.

### ۳.۲ دستور `ceph auth get-or-create` را روی کدام سرور بزنم؟

این دستور به mon های کلاستر وصل می‌شود و کاربر را همان‌جا می‌سازد. پس سروری که روی آن می‌زنید باید:

1. **عضو (یا کلاینت) همان کلاستری باشد که می‌خواهید کاربر در آن ساخته شود.**
2. فایل `/etc/ceph/ceph.conf` و یک keyring با دسترسی مدیریتی (معمولاً `/etc/ceph/ceph.client.admin.keyring`) داشته باشد.

در عمل این سرور **نود ادمین/bootstrap همان کلاستر** است (معمولاً یکی از mon ها). اگر با cephadm نصب کرده‌اید:

- روی نودی که label ‏`_admin` دارد (نودی که `cephadm bootstrap` را روی آن زده‌اید) وارد شوید.
- یا دستور را از طریق `cephadm shell -- ...` اجرا کنید.
- یا یک‌بار `cephadm install ceph-common` را روی آن نود بزنید تا دستورهای `ceph` و `rbd` را مستقیم روی خود هاست اجرا کنید.

قبل از زدن هر دستور auth، مطمئن شوید روی کلاستر درست هستید:

```bash
ceph fsid          # شناسه یکتای کلاستر؛ با fsid مورد انتظار مقایسه کنید
ceph -s            # نام و وضعیت mon ها را ببینید
hostname
```

### ۳.۳ ساخت کاربر

قالب کلی:

```bash
ceph auth get-or-create <entity> <service1> '<caps>' <service2> '<caps>' ... [-o <فایل-keyring>]
```

- `get-or-create`: اگر کاربر وجود نداشت می‌سازد؛ اگر وجود داشت و caps یکسان بود، همان را برمی‌گرداند؛ اگر caps فرق داشت **خطا می‌دهد** (برای تغییر caps از `ceph auth caps` استفاده کنید).
- `-o <فایل>`: خروجی (keyring) را در یک فایل روی **همان سروری که دستور را زده‌اید** ذخیره می‌کند. بدون `-o` فقط روی صفحه چاپ می‌شود.

**نکته مهم درباره cephadm shell:** اگر وارد `cephadm shell` تعاملی شوید و با `-o` فایل بسازید، فایل **داخل کانتینر** می‌ماند و با خروج از شل از بین می‌رود. دو راه امن:

```bash
# راه ۱: خروجی را با ریدایرکت روی خود هاست بگیرید
cephadm shell -- ceph auth get-or-create client.x mon 'allow r' osd 'allow rw' > /root/ceph.client.x.keyring

# راه ۲: ceph-common را روی هاست نصب کنید و دستور را مستقیم بزنید
cephadm install ceph-common
ceph auth get-or-create client.x mon 'allow r' osd 'allow rw' -o /root/ceph.client.x.keyring
```

(راه ۱ پیام‌های اطلاعاتی cephadm را روی stderr می‌نویسد و فایل فقط خروجی دستور را دارد.)

### ۳.۴ دستورهای مدیریتی

```bash
ceph auth ls                          # فهرست همه کاربران
ceph auth get client.x                # دیدن key و caps (به شکل keyring)
ceph auth print-key client.x          # فقط چاپ key
ceph auth caps client.x mon 'allow r' osd 'allow rw'   # تغییر caps
ceph auth export client.x > file      # خروجی گرفتن
ceph auth import -i file              # وارد کردن
ceph auth rm client.x                 # حذف
```

> `ceph auth caps` **همه caps قبلی را جایگزین می‌کند**. اگر فقط یک سرویس را تغییر می‌دهید، caps بقیه را هم دوباره بنویسید.

### ۳.۵ قراردادن keyring روی سرور مصرف‌کننده

سروری که daemon یا ابزار روی آن اجرا می‌شود باید دو چیز داشته باشد: `ceph.conf` همان کلاستر و keyring آن کاربر. مسیر پیش‌فرضی که Ceph خودکار می‌خواند:

```
/etc/ceph/<نام-کلاستر>.<entity>.keyring
مثال: /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
```

بعد از قرار دادن فایل:

```bash
chmod 600 /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
chown ceph:ceph /etc/ceph/ceph.client.rbd-mirror.site-b.keyring   # اگر daemon با کاربر ceph (بدون cephadm) اجرا می‌شود
```

تست اینکه کاربر درست کار می‌کند (بدون پیشوند `client.` در `--id`):

```bash
ceph -s --id rbd-mirror.site-b
# یا با مشخص‌کردن صریح
ceph -s --name client.rbd-mirror.site-b --keyring /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
```

### ۳.۶ انتقال امن فایل بین سرورها

- هر چیزی که شامل key یا token باشد مثل **رمز** است. از چت، ایمیل یا پیام‌رسان نفرستید.
- انتقال با `scp` (یا `rsync -e ssh`):

```bash
scp /root/فایل root@<IP-سرور-مقصد>:/root/فایل
```

- روی مقصد بلافاصله دسترسی را محدود کنید: `chmod 600 /root/فایل`.
- بعد از استفاده، فایل token را از **هر دو** سرور حذف کنید: `shred -u /root/فایل`.

### ۳.۷ اشتباهات رایج

| اشتباه | نتیجه |
|---|---|
| زدن دستور روی کلاستر اشتباه | کاربر در کلاستر دیگری ساخته می‌شود و daemon خطای auth می‌دهد |
| `ceph auth caps` بدون نوشتن همه سرویس‌ها | بعضی caps حذف می‌شوند |
| ساخت فایل داخل `cephadm shell` تعاملی | فایل بعد از خروج از بین می‌رود |
| فراموش‌کردن `chmod 600` / `chown ceph:ceph` | هشدار دسترسی یا نبود امکان خواندن keyring توسط daemon |
| کپی keyring بین دو کلاستر | کار نمی‌کند، چون کاربر در کلاستر دوم تعریف نشده؛ peering با token انجام می‌شود |

### ۳.۸ کاربران cephx در CephFS mirroring

| کاربر | روی کدام کلاستر | چه کسی می‌سازد | کاربرد |
|---|---|---|---|
| `client.admin` | هر دو | هنگام نصب | اجرای دستورهای `ceph fs snapshot mirror ...` |
| `client.cephfs-mirror.<host>.<id>` | **مبدأ** (`site-a`) | cephadm خودکار؛ در نصب دستی شما | اتصال daemon به کلاستر محلی |
| `client.mirror_remote` | **مقصد** (`site-b`) | شما با `ceph fs authorize` | اتصال daemon مبدأ به کلاستر مقصد |

نکته‌ها:

- `client.mirror_remote` را روی **site-b** می‌سازید، چون daemon مبدأ باید روی مقصد بنویسد.
- key این کاربر داخل token قرار می‌گیرد و با `peer_bootstrap import` به مبدأ می‌رسد؛ **keyring آن را دستی کپی نکنید**.

ساخت کاربر ریموت. **روی نود ادمین `site-b`:**

```bash
ceph fs authorize cephfs client.mirror_remote / rwps
```

دستور خودش caps مناسب را می‌سازد و روی صفحه چاپ می‌کند؛ نیازی به `get-or-create` دستی نیست. بررسی:

```bash
ceph auth get client.mirror_remote
```

ساخت دستی کاربر daemon (فقط بدون cephadm). **روی نود ادمین `site-a`:**

```bash
ceph auth get-or-create client.cephfs-mirror.site-a \
  mon 'profile cephfs-mirror' \
  mds 'allow r' \
  osd 'allow rw tag cephfs metadata=*, allow r tag cephfs data=*' \
  mgr 'allow r' \
  -o /etc/ceph/ceph.client.cephfs-mirror.site-a.keyring
```

سپس فایل را مطابق بخش ۳.۵ روی نودی که `cephfs-mirror` اجرا می‌شود قرار دهید و `chown ceph:ceph` و `chmod 600` را بزنید.

## ۴. فایل‌های تولیدشده و انتقال بین کلاسترها

در CephFS mirroring فقط **bootstrap token** بین کلاسترها جابه‌جا می‌شود. جهت انتقال برعکس RBD است: token روی **مقصد** ساخته می‌شود و به **مبدأ** می‌رود.

### جدول فایل‌ها

| فایل | کجا ساخته می‌شود | مسیر مبدأ | منتقل به | مسیر مقصد | بعد از انتقال |
|---|---|---|---|---|---|
| bootstrap token (JSON) | نود ادمین `site-b` (مقصد) | `/root/cephfs-token.json` | نود ادمین `site-a` (مبدأ) | `/root/cephfs-token.json` | `chmod 600`، استخراج token، import، حذف امن از هر دو سرور |
| keyring کاربر daemon (فقط بدون cephadm) | نود ادمین `site-a` | `/etc/ceph/ceph.client.cephfs-mirror.site-a.keyring` | نودی از `site-a` که daemon روی آن اجرا می‌شود | همان مسیر | `chown ceph:ceph`، `chmod 600` |

### مراحل دقیق انتقال token

**۱. روی نود ادمین `site-b`** (ساخت فایل روی خود هاست):

```bash
# اگر ceph-common روی هاست نصب است
ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json

# اگر از cephadm shell استفاده می‌کنید
cephadm shell -- ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json

chmod 600 /root/cephfs-token.json
```

**۲. انتقال به `site-a`** (از نود ادمین site-b):

```bash
scp /root/cephfs-token.json root@<IP-نود-ادمین-site-a>:/root/cephfs-token.json
```

**۳. بعد از انتقال، روی نود ادمین `site-a`:**

```bash
chmod 600 /root/cephfs-token.json

# استخراج رشته token از JSON (یکی از دو روش)
TOKEN=$(jq -r .token /root/cephfs-token.json)
# بدون jq:
TOKEN=$(python3 -c 'import json; print(json.load(open("/root/cephfs-token.json"))["token"])')

ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
# با cephadm shell (متغیر روی هاست جایگزین می‌شود):
cephadm shell -- ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
```

**۴. حذف امن token از هر دو سرور:**

```bash
unset TOKEN
shred -u /root/cephfs-token.json      # روی site-a
shred -u /root/cephfs-token.json      # روی site-b
```

**۵. بررسی:**

```bash
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror daemon status
```

### انتقال keyring (فقط بدون cephadm)

```bash
# روی نود ادمین site-a
scp /etc/ceph/ceph.client.cephfs-mirror.site-a.keyring root@<IP-نود-mirror>:/etc/ceph/
# روی نود mirror (باید ceph.conf کلاستر site-a هم آنجا باشد)
chown ceph:ceph /etc/ceph/ceph.client.cephfs-mirror.site-a.keyring
chmod 600 /etc/ceph/ceph.client.cephfs-mirror.site-a.keyring
ceph -s --id cephfs-mirror.site-a       # تست
```

## ۵. مراحل راه‌اندازی

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
- خروجی یک JSON با فیلد `token` است. آن را مثل یک رمز نگه دارید، چون دسترسی به کلاستر مقصد را می‌دهد. روش دقیق ذخیره در فایل، انتقال به site-a و مراحل بعد از آن در بخش ۴ آمده است.

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

## ۶. ساخت snapshot

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

## ۷. بررسی و تست

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

## ۸. عیب‌یابی

| علامت | بررسی |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps`، `ceph log last cephadm`، لاگ کانتینر با `cephadm logs --name cephfs-mirror.<id>` |
| peer اضافه نمی‌شود | اتصال شبکه به mon های مقصد، اعتبار token، نام filesystem |
| داده منتقل نمی‌شود | آیا snapshot ساخته شده؟ آیا مسیر با `snapshot mirror add` اضافه شده؟ |
| failed در status | `ceph fs snapshot mirror daemon status` و لاگ daemon |
| authentication error | caps کاربر `client.mirror_remote` در مقصد را با `ceph auth get` بررسی کنید |

---

## ۹. حذف و غیرفعال‌سازی

```bash
ceph fs snapshot mirror remove cephfs /data          # حذف یک دایرکتوری
ceph fs snapshot mirror peer_list cephfs             # گرفتن uuid مربوط به peer
ceph fs snapshot mirror peer_remove cephfs <uuid>    # حذف peer
ceph fs snapshot mirror disable cephfs               # غیرفعال‌کردن کامل
```

---

## ۱۰. نکات مهم

- روی دایرکتوری‌هایی که mirror می‌شوند در مقصد **ننویسید**؛ تغییرات شما در sync بعدی بازنویسی می‌شود.
- این mirroring یک‌طرفه است. برای failover باید مقصد را به‌صورت دستی به‌عنوان filesystem فعال استفاده کنید و برگشت (failback) فرآیند جدایی دارد.
- حجم زیاد داده و تعداد زیاد فایل‌های کوچک، sync را کند می‌کند.
- برای backup واقعی (برگشت به نقطه زمانی)، snapshot های retention دار و در صورت نیاز ابزار backup جدا هم نگه دارید.
