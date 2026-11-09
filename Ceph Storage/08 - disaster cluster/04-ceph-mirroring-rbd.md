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

### ۳.۸ کاربران cephx در RBD mirroring

| کاربر | روی کدام کلاستر | چه کسی می‌سازد | کاربرد | caps |
|---|---|---|---|---|
| `client.admin` | هر دو | هنگام نصب | اجرای دستورهای `rbd mirror pool ...` | کامل |
| `client.rbd-mirror.<id>` | کلاستری که daemon در آن اجرا می‌شود (در حالت یک‌طرفه: `site-b`) | cephadm خودکار؛ در نصب دستی شما | اتصال daemon به کلاستر **محلی** خودش | `mon 'profile rbd-mirror' osd 'profile rbd'` |
| `client.rbd-mirror-peer` | کلاستری که token را می‌سازد (`site-a`) | **خودکار** هنگام `bootstrap create` | اتصال daemon کلاستر مقابل به این کلاستر | `mon 'profile rbd-mirror-peer' osd 'profile rbd'` |

نکته‌ها:

- با cephadm لازم نیست `client.rbd-mirror.<id>` را دستی بسازید؛ `ceph orch apply rbd-mirror` آن را می‌سازد.
- اطلاعات اتصال به کلاستر مقابل داخل token است و بعد از `bootstrap import` در تنظیمات mon کلاستر ذخیره می‌شود، پس **keyring ی بین دو کلاستر کپی نمی‌شود**.

ساخت دستی کاربر daemon (فقط بدون cephadm). **روی نود ادمین `site-b`:**

```bash
ceph auth get-or-create client.rbd-mirror.site-b \
  mon 'profile rbd-mirror' osd 'profile rbd' \
  -o /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
```

اگر نودی که daemon روی آن اجرا می‌شود **همان نود ادمین نیست**، فایل را به آن نود منتقل کنید (جدول بخش ۴).

بررسی:

```bash
ceph auth get client.rbd-mirror.site-b      # روی site-b
ceph auth get client.rbd-mirror-peer        # روی site-a، بعد از bootstrap create
```

## ۴. فایل‌های تولیدشده و انتقال بین کلاسترها

در RBD mirroring فقط **یک فایل** بین کلاسترها جابه‌جا می‌شود: bootstrap token. (keyring کاربر daemon فقط در نصب بدون cephadm و فقط بین نودهای **همان کلاستر** منتقل می‌شود.)

### جدول فایل‌ها

| فایل | کجا ساخته می‌شود | مسیر مبدأ | منتقل به | مسیر مقصد | بعد از انتقال |
|---|---|---|---|---|---|
| bootstrap token | نود ادمین `site-a` | `/root/rbd-token` | نود ادمین `site-b` | `/root/rbd-token` | `chmod 600`، import، حذف امن از هر دو سرور |
| keyring کاربر daemon (فقط بدون cephadm) | نود ادمین `site-b` | `/etc/ceph/ceph.client.rbd-mirror.site-b.keyring` | نودی از `site-b` که daemon روی آن اجرا می‌شود | همان مسیر | `chown ceph:ceph`، `chmod 600`، راه‌اندازی سرویس |

### مراحل دقیق انتقال token

**۱. روی نود ادمین `site-a`** (ساخت فایل روی خود هاست):

```bash
# اگر ceph-common روی هاست نصب است
rbd mirror pool peer bootstrap create --site-name site-a rbd > /root/rbd-token

# اگر از cephadm shell استفاده می‌کنید
cephadm shell -- rbd mirror pool peer bootstrap create --site-name site-a rbd > /root/rbd-token

chmod 600 /root/rbd-token
```

**۲. انتقال به `site-b`** (از نود ادمین site-a):

```bash
scp /root/rbd-token root@<IP-نود-ادمین-site-b>:/root/rbd-token
```

**۳. بعد از انتقال، روی نود ادمین `site-b`:**

```bash
chmod 600 /root/rbd-token
rbd mirror pool peer bootstrap import --site-name site-b --direction rx-only rbd /root/rbd-token
```

اگر فقط `cephadm shell` دارید، فایل را داخل کانتینر در دسترس بگذارید (فایل داخل کانتینر در مسیر `/mnt/<نام-فایل>` ظاهر می‌شود؛ با `ls /mnt` تأیید کنید):

```bash
cephadm shell --mount /root/rbd-token -- \
  rbd mirror pool peer bootstrap import --site-name site-b --direction rx-only rbd /mnt/rbd-token
```

**۴. حذف امن token از هر دو سرور** (بعد از موفقیت import):

```bash
shred -u /root/rbd-token      # روی site-b
shred -u /root/rbd-token      # روی site-a
```

**۵. بررسی:**

```bash
rbd mirror pool info rbd           # باید peer را با site-name نشان بدهد
rbd mirror pool status rbd --verbose
```

### انتقال keyring (فقط بدون cephadm)

```bash
# روی نود ادمین site-b
scp /etc/ceph/ceph.client.rbd-mirror.site-b.keyring root@<IP-نود-mirror>:/etc/ceph/
# روی نود mirror (باید ceph.conf کلاستر site-b هم آنجا باشد)
chown ceph:ceph /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
chmod 600 /etc/ceph/ceph.client.rbd-mirror.site-b.keyring
ceph -s --id rbd-mirror.site-b          # تست
systemctl enable --now ceph-rbd-mirror@rbd-mirror.site-b
```

## ۵. مراحل راه‌اندازی (یک‌طرفه: site-a به site-b)

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

فایل token را امن به site-b منتقل کنید (مثلاً با scp). مثل رمز با آن رفتار کنید. جزئیات کامل مسیرها و مراحل بعد از انتقال در بخش ۴ آمده است.

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

## ۶. بررسی و تست

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

## ۷. Failover و Failback

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

## ۸. عیب‌یابی

| علامت | بررسی |
|---|---|
| daemon نیست | `ceph orch ps`، `cephadm logs --name rbd-mirror.<id>` |
| `down+unknown` | شبکه، اجرای daemon، اعتبار کاربر peer |
| `up+error` | `rbd mirror image status rbd/img1` و بخش description |
| split-brain بعد از force promote | resync با `rbd mirror image resync` روی طرف قدیمی |
| replay کند | حجم نوشتن، سرعت شبکه، منابع نود mirror |

---

## ۹. حذف و غیرفعال‌سازی

```bash
rbd mirror image disable rbd/img1
rbd mirror pool disable rbd
rbd mirror pool peer remove rbd <peer-uuid>
```

---

## ۱۰. نکات مهم

- Mirroring برابر با backup نیست. برای برگشت به نقطه زمانی از snapshot های دوره‌ای یا `rbd export-diff` استفاده کنید.
- در حالت یک‌طرفه، failback خودکار وجود ندارد.
- حالت journal برای نوشتن سنگین، سربار قابل‌توجه دارد؛ در صورت نیاز به RPO نه‌چندان کم، snapshot-based ساده‌تر است.
- همیشه بعد از هر تغییر، `rbd mirror pool status --verbose` را بررسی کنید.
