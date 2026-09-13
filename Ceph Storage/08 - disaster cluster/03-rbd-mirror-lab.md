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

هر سایت باید conf و کلید سایت مقابل را داشته باشد.

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

روی **هر دو** کلاستر، برای Pool `rbd-pool-app1` و در حالت pool:

```bash
rbd mirror pool enable rbd-pool-app1 pool
```

اگر از قبل enable شده باشد:

```text
rbd: mirroring is already configured for pool mode
```

## ۶. اضافه کردن Peer

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

```bash
rbd mirror pool info rbd-pool-app1
rbd mirror pool peer remove rbd-pool-app1 431f522d-dcd3-4450-804d-b845e1b70bad
```

تا وقتی peer ثبت است، disable کردن mirroring Pool شکست می‌خورد:

```text
peers still registered
```

## ۱۰. خطاهای رایج این لاب

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
