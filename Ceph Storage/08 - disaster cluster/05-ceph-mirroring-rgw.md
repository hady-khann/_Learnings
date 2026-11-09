# مستند راه‌اندازی Multisite برای RGW (Object Storage)

## ۱. مرور کلی

در RGW به‌جای کلمه mirror از **multisite** استفاده می‌شود. ساختار منطقی:

- **Realm**: بالاترین سطح، یک فضای نام سراسری.
- **Zonegroup**: گروهی از zone ها (قبلاً region). فقط یک zonegroup عضو **master** است.
- **Zone**: یک کلاستر Ceph (یا گروهی از RGW ها روی یک کلاستر). هر zonegroup یک zone **master** دارد.
- **Period**: نسخه‌ای از پیکربندی realm؛ با `period update --commit` اعمال می‌شود.

رفتار: به‌صورت پیش‌فرض **active-active** است؛ داده‌ها (آبجکت‌ها) بین zone ها دوطرفه sync می‌شوند. عملیات metadata (ساخت کاربر و bucket) از طریق zone master می‌گذرد.

### فرضیات

| مورد | مقدار |
|---|---|
| realm | `myrealm` |
| zonegroup | `mygroup` |
| zone اول (master، کلاستر site-a) | `zone-a` با endpoint `http://site-a-rgw:8080` |
| zone دوم (secondary، کلاستر site-b) | `zone-b` با endpoint `http://site-b-rgw:8080` |
| روش نصب | cephadm (Quincy یا جدیدتر) |

> دستورها بر اساس مستندات رسمی Ceph است. قبل از production روی لب تست کنید. اگر روی کلاستر از قبل RGW با realm و zone پیش‌فرض (`default`) اجرا می‌شود، این روند روی کلاستر تازه ساده‌تر است و برای کلاستر دارای داده باید مهاجرت جداگانه انجام شود.

---

## ۲. پیش‌نیازها

روی هر دو کلاستر:

```bash
ceph -s           # HEALTH_OK
ceph versions
```

- هر endpoint RGW باید از کلاستر طرف مقابل در دسترس باشد (تست با `curl http://site-b-rgw:8080`).
- ساعت‌ها با chrony/NTP هماهنگ باشد (اختلاف زمان باعث خطای احراز هویت S3 می‌شود).
- در محیط production از HTTPS و endpoint های پایدار (مثلاً پشت load balancer) استفاده کنید.

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

### ۳.۸ cephx و کلید S3 در RGW (دو چیز متفاوت)

| مفهوم | کجا تعریف می‌شود | کاربرد | چه کسی می‌سازد |
|---|---|---|---|
| کاربر cephx برای daemon RGW، مثل `client.rgw.<service>.<host>.<id>` | **هر کلاستر برای خودش** | اتصال daemon RGW به کلاستر Ceph محلی | cephadm خودکار؛ در نصب دستی شما |
| **system user** (`sync-user`) با access/secret key | در دیتابیس RGW، روی کلاستر master | ارتباط zone ها با هم از طریق API خود RGW | شما با `radosgw-admin user create --system` |

نکته مهم: **برخلاف RBD و CephFS، در RGW هیچ keyring یا token ای بین کلاسترها منتقل نمی‌شود.** فقط access key و secret key کاربر `sync-user` باید به کلاستر دوم برسد (با آن `realm pull` و `period pull` و `zone create` انجام می‌شود).

دو کاربر cephx در دو کلاستر **بدون هیچ ارتباطی** با هم هستند و کپی نمی‌شوند.

ساخت دستی کاربر cephx برای RGW (فقط بدون cephadm). **روی نود ادمین کلاستر مربوطه** (مثلاً `site-a`):

```bash
ceph auth get-or-create client.rgw.zone-a.host1 \
  mon 'allow rw' osd 'allow rwx' mgr 'allow rw' \
  -o /etc/ceph/ceph.client.rgw.zone-a.host1.keyring
```

بعد فایل را روی نودی که RGW اجرا می‌شود قرار دهید (بخش ۳.۵) و در `ceph.conf` همان نود بخش `[client.rgw.zone-a.host1]` با `rgw_zone` و `rgw_frontends` را بنویسید.

بررسی کاربران RGW:

```bash
ceph auth ls | grep rgw
radosgw-admin user list
```

## ۴. فایل‌های تولیدشده و انتقال بین کلاسترها

در RGW multisite تنها داده حساسی که به کلاستر دوم می‌رود، **کلیدهای کاربر `sync-user`** است. بهتر است خروجی ساخت کاربر را در فایل ذخیره کنید و به‌جای تایپ دستی، از روی همان فایل بخوانید.

### جدول فایل‌ها

| فایل | کجا ساخته می‌شود | مسیر مبدأ | منتقل به | مسیر مقصد | بعد از انتقال |
|---|---|---|---|---|---|
| خروجی کاربر `sync-user` (JSON، شامل access/secret key) | نود ادمین `site-a` | `/root/sync-user.json` | نود ادمین `site-b` | `/root/sync-user.json` | `chmod 600`، خواندن کلیدها در متغیر، اجرای دستورات، حذف امن از هر دو سرور |

### مراحل دقیق

**۱. روی نود ادمین `site-a`:**

```bash
cephadm shell -- radosgw-admin user create --uid=sync-user --display-name="Sync User" --system > /root/sync-user.json
chmod 600 /root/sync-user.json

# خواندن کلیدها برای ثبت در zone
ACCESS=$(jq -r '.keys[0].access_key' /root/sync-user.json)
SECRET=$(jq -r '.keys[0].secret_key' /root/sync-user.json)
cephadm shell -- radosgw-admin zone modify --rgw-zone=zone-a --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin period update --commit
```

(متغیرها روی هاست جایگزین می‌شوند و قبل از رسیدن به کانتینر مقدار واقعی دارند. اگر `jq` نصب نیست، `apt install jq` یا کلیدها را از خروجی JSON کپی کنید.)

**۲. انتقال به `site-b`:**

```bash
scp /root/sync-user.json root@<IP-نود-ادمین-site-b>:/root/sync-user.json
```

**۳. بعد از انتقال، روی نود ادمین `site-b`:**

```bash
chmod 600 /root/sync-user.json
ACCESS=$(jq -r '.keys[0].access_key' /root/sync-user.json)
SECRET=$(jq -r '.keys[0].secret_key' /root/sync-user.json)

cephadm shell -- radosgw-admin realm pull --url=http://site-a-rgw:8080 --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin realm default --rgw-realm=myrealm
cephadm shell -- radosgw-admin period pull --url=http://site-a-rgw:8080 --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-b \
  --endpoints=http://site-b-rgw:8080 --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin period update --commit
```

**۴. حذف امن فایل و متغیرها از هر دو سرور:**

```bash
unset ACCESS SECRET
shred -u /root/sync-user.json      # روی site-b
shred -u /root/sync-user.json      # روی site-a
```

**۵. بررسی:**

```bash
cephadm shell -- radosgw-admin sync status
```

> اگر به‌جای `cephadm shell --` مستقیم `radosgw-admin` را می‌زنید، بسته `radosgw` باید روی آن نود نصب باشد.

## ۵. مراحل راه‌اندازی

### بخش الف: کلاستر اول، zone master (site-a)

**مرحله ۱: ساخت realm**

```bash
radosgw-admin realm create --rgw-realm=myrealm --default
```

**مرحله ۲: ساخت zonegroup (master)**

```bash
radosgw-admin zonegroup create --rgw-zonegroup=mygroup \
  --endpoints=http://site-a-rgw:8080 --master --default
```

**مرحله ۳: ساخت zone (master)**

```bash
radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-a \
  --endpoints=http://site-a-rgw:8080 --master --default
```

**مرحله ۴: ساخت system user**

```bash
radosgw-admin user create --uid=sync-user --display-name="Sync User" --system
```

خروجی شامل `access_key` و `secret_key` است. آن‌ها را یادداشت کنید.

**مرحله ۵: ثبت کلیدها در zone و commit کردن period**

```bash
radosgw-admin zone modify --rgw-zone=zone-a \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
radosgw-admin period update --commit
```

**مرحله ۶: اجرای RGW برای zone-a**

```bash
ceph orch apply rgw mysite --realm=myrealm --zone=zone-a \
  --placement=1 --port=8080
ceph orch ps | grep rgw
```

### بخش ب: کلاستر دوم، zone secondary (site-b)

**مرحله ۱: pull کردن realm و period از master**

```bash
radosgw-admin realm pull --url=http://site-a-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
radosgw-admin realm default --rgw-realm=myrealm
radosgw-admin period pull --url=http://site-a-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
```

**مرحله ۲: ساخت zone ثانویه**

```bash
radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-b \
  --endpoints=http://site-b-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
```

**مرحله ۳: commit کردن period**

```bash
radosgw-admin period update --commit
```

**مرحله ۴: اجرای RGW برای zone-b**

```bash
ceph orch apply rgw mysite --realm=myrealm --zone=zone-b \
  --placement=1 --port=8080
```

اگر لازم شد RGW ها را restart کنید:

```bash
ceph orch restart rgw.mysite
```

---

## ۶. بررسی و تست

وضعیت sync روی هر دو کلاستر:

```bash
radosgw-admin sync status
radosgw-admin period get
radosgw-admin zonegroup get
```

خروجی سالم `sync status` چیزی شبیه این نشان می‌دهد: metadata sync و data sync هر دو `caught up`.

تست عملی:

```bash
# روی site-a: ساخت کاربر عادی
radosgw-admin user create --uid=testuser --display-name="Test User"

# ساخت bucket و آپلود با aws cli (به endpoint مبدأ)
aws --endpoint-url http://site-a-rgw:8080 s3 mb s3://testbucket
aws --endpoint-url http://site-a-rgw:8080 s3 cp file.txt s3://testbucket/

# چند ثانیه بعد، روی zone-b
aws --endpoint-url http://site-b-rgw:8080 s3 ls s3://testbucket/
radosgw-admin bucket list
```

برای aws cli از access_key و secret_key خروجی `testuser` استفاده کنید (با `aws configure`).

---

## ۷. Failover (از دست رفتن master)

اگر zone master از دسترس خارج شد و می‌خواهید zone ثانویه را master کنید (روی site-b):

```bash
radosgw-admin zone modify --rgw-zone=zone-b --master --default
radosgw-admin period update --commit
```

بعد از برگشت master قدیمی، باید آن را با period جدید sync کنید (pull دوباره) و سپس در صورت نیاز نقش‌ها را برگردانید. این فرآیند را قبل از production روی لب تمرین کنید.

---

## ۸. عیب‌یابی

| علامت | بررسی |
|---|---|
| `realm pull` یا `period pull` خطا می‌دهد | دسترسی شبکه به endpoint master، صحت access/secret، ساعت سرورها |
| `sync status` نشان‌دهنده behind | شبکه، لاگ RGW، `radosgw-admin sync error list` |
| خطای 403 در sync | کلیدهای system user در `zone modify` ثبت شده؟ آیا `period update --commit` اجرا شده؟ |
| کاربر یا bucket ساخته‌شده روی secondary دیده نمی‌شود | metadata فقط از master می‌آید؛ sync status را بررسی کنید |
| RGW بالا نمی‌آید | `ceph orch ps`، لاگ کانتینر با `cephadm logs --name rgw.<id>` |

---

## ۹. نکات مهم

- RGW multisite به‌طور پیش‌فرض **دوطرفه** است؛ بعد از ساخت zone ثانویه می‌توانید روی هر دو zone آبجکت بنویسید. اما ساخت کاربر و bucket از طریق master متادیتا را مدیریت می‌کند.
- این مکانیزم برابر با backup نیست: حذف آبجکت هم sync می‌شود. برای محافظت در برابر حذف از **versioning** و lifecycle policy استفاده کنید.
- پیکربندی realm/zonegroup/zone حساس است و هر تغییر باید با `period update --commit` اعمال شود.
- برای ترافیک بین zone ها از HTTPS استفاده کنید و کلیدهای system user را امن نگه دارید.
