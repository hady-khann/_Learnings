# Poolهای OpenStack، کاربر CephX، و secret در libvirt

نیمهٔ عملی جلسهٔ نهم روی کلاستر **hoodadcloud** است، نه VMهای آموزشی `192.168.1.x`. Horizon روی `hoodadcloud.ir` است. نودها:

| نقش | Hostname | نکته |
|---|---|---|
| Ceph | `ceph-1` | cephadm / image `quay.io/ceph/ceph` |
| OpenStack controller | `controller` | Glance، Keystone، `openstack` CLI |
| Nova compute | `compute` | `virsh` اینجا نصب است، نه روی controller |

MONها در `ceph.conf`:

```text
# minimal ceph.conf for 9ce9724a-6bc1-11ed-8cc5-1e5063c36adf
[global]
fsid = 9ce9724a-6bc1-11ed-8cc5-1e5063c36adf
mon_host = [v2:185.55.227.16:3300/0,v1:185.55.227.16:6789/0] [v2:185.55.227.42:3300/0,v1:185.55.227.42:6789/0] [v2:185.55.227.141:3300/0,v1:185.55.227.141:6789/0]
```

روی `ceph-1` در `/etc/ceph` فقط `ceph.conf`، `ceph.keyring`، `rbdmap` بود. دستور `clear` نصب نبود.

Glance و ساخت image در `02-glance-and-nova.md` است. HAProxy روی همین نود در `15 - HAProxy`.

## ۱. کپی `ceph.conf` به controller

از `ceph-1`:

```bash
cd /etc/ceph
scp ceph.conf root@controller:/etc/ceph/ceph.conf
```

اولین تلاش مسیر را ناقص گذاشت (`/etc/ce`)؛ باید فایل کامل مقصد را بدهید.

## ۲. ساخت Pool برای Cinder و Nova

روی `ceph-1`:

```bash
ceph osd pool volumes 128
```

خطا: `no valid command found` — کلمهٔ `create` جا افتاده بود.

```bash
ceph osd pool create volumes 128
```

خروجی: `pool 'volumes' created`.

```bash
ceph osd pool create vms 128
```

خطا:

```text
Error ERANGE: pg_num 128 size 3 would mean 1761 total pgs, which exceeds max 1500 (mon_max_pg_per_osd 250 * num_in_osds 6)
```

یعنی ۶ OSD in و سقف ۱۵۰۰ PG. بعد `pg_num` را پایین آوردند (`ceph osd pool create vms 20`). Pool به نام `images` از قبل برای Glance وجود داشت (`rbd ls images` بعداً image را نشان داد).

## ۳. کاربر `client.cinder`

اول فقط Pool `volumes`:

```bash
ceph auth get-or-create client.cinder mon 'allow *' osd 'allow * pool=volumes'
```

خروجی لاب:

```text
[client.cinder]
        key = AQDVg4NjT5Z9MxAA36YKq91rgZjlC0nzLrganw==
```

بعد خواستند `vms` و `images` را هم به cap اضافه کنند:

```bash
ceph auth get-or-create client.cinder mon 'allow *' osd 'allow * pool=volumes, allow * pool=vms allow * pool=images'
```

```text
Error EINVAL: osd capability parse failed, stopped at 'allow * pool= images'
```

ویرگول و فاصله بین capها باید جدا و کامل باشد. نمونهٔ درست (چند pool در یک رشتهٔ osd):

```bash
ceph auth get-or-create client.cinder \
  mon 'allow *' \
  osd 'allow * pool=volumes, allow * pool=vms, allow * pool=images'
```

`get-or-create` اگر کاربر از قبل باشد کلید قبلی را برمی‌گرداند و cap را عوض نمی‌کند؛ برای عوض کردن cap از `ceph auth caps` استفاده کنید.

## ۴. کاربر `client.glance` و ریختن keyring روی controller

```bash
ceph auth get-or-create client.glance | ssh root@controller sudo tee /etc/ceph/ceph.client.glance.keyring
```

اولین pipe مسیر را `/etc/ce` برید. بار دوم فایل درست نوشته شد:

```text
[client.glance]
        key = AQAAAhYNjJS4pJRAA+dueCeKu9HZ0T0+YAf8GFw==
```

روی controller بعداً کلید Cinder را هم گرفتند:

```bash
ceph auth get-key client.cinder | ssh root@controller tee /etc/ceph/temp.cli
```

لیست `/etc/ceph` روی controller:

```text
ceph.client.cinder.keyring  ceph.conf  temp.cli
ceph.client.glance.keyring  rbdmap
```

## ۵. `--name` باید id باشد، نه اسم فایل

```bash
ceph -s --name ceph.client.cinder.keyring
```

```text
Error initializing cluster client: rados_initialize failed with error code: -22
```

`--name` مقدار CephX است (`client.cinder`)، نه مسیر keyring. keyring را با `-k` یا گذاشتن فایل در `/etc/ceph` بدهید.

## ۶. `secret.xml` برای libvirt

روی **controller** در `/etc/ceph`:

```bash
uuidgen
```

خروجی لاب:

```text
82b2ce11-e3d0-4b8e-9507-67a994eaa834
```

```bash
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>82b2ce11-e3d0-4b8e-9507-67a994eaa834</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```

```bash
virsh secret-define --file secret.xml
```

```text
bash: virsh: command not found
```

`virsh` روی controller نبود. فایل را به compute بردند و آنجا تعریف کردند.

کپی keyring خام:

```bash
scp temp.client.cinder.keyring root@compute:/etc/temp.client.cinder.keyring
```

روی compute یک‌بار `cd /etc/secret.xml` زدند — فایل است، پوشه نیست.

## ۷. تعریف secret روی compute

UUIDیی که `virsh secret-list` نشان داد با `uuidgen` روی controller یکی نبود:

```text
UUID                                  Usage
----------------------------------------------------------
19deff5b-d6cf-4026-b2c7-34892863393a  ceph client.cinder secret
```

یعنی روی compute یا secret از قبل بود، یا `secret.xml` را با UUID دیگری ساختند. Nova/Cinder باید **همین** UUID را در کانفیگ ببینند، نه لزوماً رقم `uuidgen` روی controller.

اول `$(cat temp.client.cinder.key)` — فایل وجود نداشت. بعد با خود keyring:

```bash
virsh secret-set-value --secret 19deff5b-d6cf-4026-b2c7-34892863393a \
  --base64 $(cat temp.client.cinder.keyring) \
  && rm temp.client.cinder.keyring secret.xml
```

هشدار libvirt:

```text
error: Passing secret value as command-line argument is insecure!
Secret value set
```

با وجود کلمهٔ `error`، مقدار set شد. `virsh secret-list` همان UUID را نشان داد.

از controller به compute باید `ceph.conf` و هر دو keyring هم برسند:

```bash
scp ceph.client.cinder.keyring ceph.client.glance.keyring compute:/etc/ceph/
```

یک‌بار مقصد را `ssh compute` نوشتند و `ssh: No such file or directory` گرفتند. روی خود compute اول فقط `rbdmap` در `/etc/ceph` بود.

> **هشدار:** UUID داخل `secret.xml` روی controller (`82b2ce11-…`) با secret واقعی compute (`19deff5b-…`) فرق داشت. اگر Nova هنوز UUID اول را در `nova.conf` / `cinder.conf` دارد، attach دیسک RBD شکست می‌خورد.
