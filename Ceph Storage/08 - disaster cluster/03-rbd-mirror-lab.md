<div dir="rtl" lang="fa">

# راه‌اندازی RBD Mirror بین دو کلاستر

پس از بالا آمدن کلاستر Passive (`backup` روی `ceph-node5/6/7`)، سایت Active (`ceph` روی `ceph-node1/2/3`) را به آن وصل کنید تا Pool به نام `rbd-pool-app1` mirror شود.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| `data` | `rbd-pool-app1` |
| `image-1` … `image-4` | `rbd-image-app1` … `rbd-image-app4` |
| `anisa-1` / `anisa-2` | `rbd-image-app5` / `rbd-image-app6` |

## ۱. ساخت کاربر روی کلاستر Disaster

دو کلاستر به‌صورت پیش‌فرض به هم اعتماد ندارند. برای mirror، هر سایت یک کاربر CephX مخصوص ارتباط بین‌سایتی می‌سازد:

| کاربر | ساخته‌شده روی | نقش |
|---|---|---|
| `client.remote` | کلاستر `backup` (Passive) | هویت daemon `rbd-mirror` روی سایت DR |
| `client.local` | کلاستر `ceph` (Active) | هویتی که سایت Passive با آن به Active وصل می‌شود |

بدون این کلیدها، daemon نمی‌تواند journal را از سایت مقابل بخواند.

روی `ceph-node5`:

```bash
ceph auth get-or-create client.remote \
  mon 'allow *' \
  osd 'allow *' \
  -o /etc/ceph/backup.client.remote.keyring
```

خروجی باید در `/etc/ceph` دیده شود:

```text
backup.conf
backup.client.admin.keyring
backup.client.remote.keyring
```

## ۲. ساخت کاربر روی کلاستر Active

روی `ceph-node1`:

```bash
ceph auth get-or-create client.local \
  mon 'allow *' \
  osd 'allow *' \
  -o /etc/ceph/ceph.client.local.keyring
```

نمونهٔ keyring:

```text
[client.local]
    key = AQBTDXBjwB7KERAATFihgxN5My6X5gOxT3e6Ew==
```

## ۳. جابه‌جا کردن conf و keyring بین دو سایت

هر سایت باید conf و کلید سایت مقابل را داشته باشد. `ceph.conf` / `backup.conf` آدرس MONها را می‌گوید؛ keyring ثابت می‌کند شما همان کاربر هستید. اگر فقط کلید را بفرستید و conf نرود، CLI نمی‌داند MON سایت مقابل کجاست. جهت هر دو طرف لازم است چون بعداً ممکن است از Active هم به Passive دستور بزنید (و برعکس).

از primary به disaster:

```bash
scp /etc/ceph/ceph.client.local.keyring \
  root@ceph-node5:/etc/ceph/ceph.client.local.keyring
```

از disaster به primary (فایل‌های `backup.conf` و `backup.client.remote.keyring`) را هم کپی کنید تا `ceph-node1` بتواند به کلاستر `backup` وصل شود.

روی `ceph-node1` در `/etc/ceph` باید چیزی شبیه این باشد:

```text
backup.client.remote.keyring
backup.conf
ceph.client.admin.keyring
ceph.client.local.keyring
ceph.conf
rbdmap
```

## ۴. نصب و روشن کردن daemon روی سایت Passive

**فرآیند `rbd-mirror` چیست؟** یک daemon جدا از MON/OSD. کارش این است که به peer وصل شود، journal Imageهای primary را بخواند، و همان نوشتن‌ها را روی Imageهای محلی **replay** کند. در مدل Active/Passive این لاب، daemon را روی سایت Passive روشن می‌کنند چون کپی باید *اینجا* ساخته شود. بدون این فرآیند، peer ثبت می‌شود ولی هیچ داده‌ای جابه‌جا نمی‌شود.

روی `ceph-node5`:

```bash
apt install -y rbd-mirror
```

سرویس instance با نام `remote` (یعنی `--id remote` / `client.remote`):

```bash
systemctl enable ceph-rbd-mirror@remote
systemctl start ceph-rbd-mirror@remote.service
systemctl status ceph-rbd-mirror@remote.service
```

فرآیند در ویدیو این‌طور اجرا می‌شد:

```text
/usr/bin/rbd-mirror -f --cluster backup --id remote
```

یعنی daemon روی کلاستر محلی `backup` با هویت `client.remote` کار می‌کند.

## ۵. Enable کردن mirroring روی Pool

این دستور فقط پرچم Pool را روشن می‌کند: «Imageهای این Pool *می‌توانند* mirror شوند.» هنوز نمی‌گوید سایت مقابل کیست. باید روی **هر دو** کلاستر اجرا شود تا هر دو طرف نقش mirror را بپذیرند؛ وگرنه یک سمت Image می‌سازد و سمت دیگر replica را قبول نمی‌کند.

روی **هر دو** کلاستر، برای Pool `rbd-pool-app1` و در حالت pool:

```bash
rbd mirror pool enable rbd-pool-app1 pool
```

اگر از قبل enable شده باشد:

```text
rbd: mirroring is already configured for pool mode
```

## ۶. اضافه کردن Peer

**همتا (Peer) چیست؟** کلاستر دوری که این Pool با آن replicate می‌شود. دو کلاستر Ceph همدیگر را نمی‌شناسند تا شما صریحاً طرف مقابل را ثبت کنید.

- دستور `rbd mirror pool enable` فقط می‌گوید این Pool *اجازه* دارد mirror شود (مثل `git init`).
- دستور `rbd mirror pool peer add` می‌گوید *به کجا* وصل شو (مثل `git remote add`). بدون peer، daemon `rbd-mirror` مبدأیی برای pull ندارد.

شکل `client.local@ceph` یعنی: با کاربر CephX به نام `client.local` به کلاستری که نامش `ceph` است وصل شو. سمت چپ هویت است، سمت راست نام Cluster سایت Active — نه hostname نود.

خروجی یک **UUID** است؛ شناسهٔ همین peer در تنظیمات Pool. برای حذف بعدی (`peer remove`) همین UUID لازم است، نه اسم `ceph`.

در خروجی `pool info`، `Direction: rx-tx` یعنی لینک *می‌تواند* داده بفرستد و بگیرد. در این لاب مدل Active/Passive است: نوشتن فقط روی Imageهای **primary** سایت Active انجام می‌شود؛ Passive replica را replay می‌کند. `rx-tx` قابلیت لینک است، نه اینکه هر دو سایت همزمان بنویسند.

روی سایت Passive (`ceph-node5`)، peer را به کلاستر Active وصل کنید:

```bash
rbd mirror pool peer add rbd-pool-app1 client.local@ceph
```

خروجی یک UUID است، مثلاً:

```text
431f522d-dcd3-4450-804d-b845e1b70bad
```

بررسی:

```bash
rbd mirror pool info rbd-pool-app1
```

نمونهٔ خروجی لاب:

```text
Mode: pool
Site Name: f5efec14-2866-48f2-bfc7-01ee9ef11358
Peer Sites:
  UUID: 431f522d-dcd3-4450-804d-b845e1b70bad
  Name: ceph
  Direction: rx-tx
  Client: client.local
```

همیشه Pool را نام ببرید. بدون `-p` / نام Pool، خطا می‌دهد:

```text
rbd: error opening default pool 'rbd'
```

## ۷. ساخت Image روی سایت Active

روی `ceph-node1`:

```bash
rbd create rbd-image-app1 \
  --size 1024 \
  --pool rbd-pool-app1 \
  --image-feature exclusive-lock,journaling

rbd create rbd-image-app2 \
  --size 1024 \
  --pool rbd-pool-app1 \
  --image-feature exclusive-lock,journaling
```

لیست:

```bash
rbd -p rbd-pool-app1 ls
```

```text
rbd-image-app1
rbd-image-app2
```

در ادامهٔ لاب Imageهای `rbd-image-app3` و `rbd-image-app4` و همچنین `rbd-image-app5` / `rbd-image-app6` هم ساخته شدند.

## ۸. وضعیت Mirror

**بازپخش (replay) یعنی** daemon journal سایت Active را می‌خواند و همان نوشتن‌ها را روی کپی محلی اجرا می‌کند. `starting replay` یعنی کار شروع شده؛ `up+replaying` یعنی daemon زنده است و همگام‌سازی ادامه دارد. `up` = فرآیند بالاست، `replaying` = در حال اعمال journal.

نسخهٔ Image سمت Active **primary** است (قابل نوشتن). کپی سمت Passive **non-primary** است؛ کلاینت نباید مستقیم روی آن بنویسد.

روی سایت Passive:

```bash
rbd mirror pool status rbd-pool-app1
```

وقتی replay شروع شده باشد:

```text
health: WARNING
images: 2 total  2 starting replay
```

بعد از همگام شدن Image:

```bash
rbd mirror image status rbd-pool-app1/rbd-image-app5
```

```text
state: up+replaying
service: remote on ceph-node5
```

و همان Imageها روی disaster دیده می‌شوند:

```bash
rbd -p rbd-pool-app1 ls
```

```text
rbd-image-app5
rbd-image-app6
```

اگر daemon گیر کرد:

```bash
systemctl restart ceph-rbd-mirror@remote.service
```

در `ceph -s` کلاستر disaster باید سرویس `rbd-mirror` ظاهر شود:

```text
rbd-mirror: 1 daemon active
```

## ۹. حذف Peer (در صورت نیاز)

تا وقتی peer ثبت است، Pool هنوز به سایت مقابل گره خورده. برای همین `mirror pool disable` با خطای `peers still registered` رد می‌شود: اول رابطه را قطع کنید، بعد mirroring را خاموش کنید.

```bash
rbd mirror pool info rbd-pool-app1
rbd mirror pool peer remove rbd-pool-app1 431f522d-dcd3-4450-804d-b845e1b70bad
```

تا وقتی peer ثبت است، disable کردن mirroring Pool شکست می‌خورد:

```text
peers still registered
```

## ۱۰. خطاهای رایج این لاب

چون کپی Passive غیرقابل‌نوشتن (non-primary) است، `rbd rm` معمولی آن را پاک نمی‌کند — Ceph فرض می‌کند هنوز به primary وصل است. `force` یعنی «می‌دانم این replica است، قطعش کن»؛ اگر اشتباه روی دادهٔ زنده بزنید، mirror خراب می‌شود.

حذف Image روی سایت Passive (نسخهٔ non-primary) بدون force:

```bash
rbd rm -p rbd-pool-app1 rbd-image-app4
```

```text
mirrored image is not primary, add force option to disable mirroring
rbd: delete error: (22) Invalid argument
```

خاموش کردن mirroring کل Pool وقتی هنوز Imageهای replica هستند:

```bash
rbd mirror pool disable rbd-pool-app1
```

همان خطای «not primary / add force» را می‌دهد.

> **هشدار:** Image سمت Passive فقط replica است. حذف یا disable را روی primary انجام دهید، یا صریحاً با `force` جلو بروید — در غیر این صورت دادهٔ mirror خراب می‌شود.

اگر بعد از reboot نود disaster وضعیت `unknown` ماند، سرویس `ceph-rbd-mirror@remote` و `ceph -s --cluster backup` را دوباره چک کنید.
</div>
