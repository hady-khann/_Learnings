<div dir="rtl" lang="fa">

# Mount کردن CephFS با Kernel Client

روی `ceph-node1` کاربر `client.fs` بسازید، keyring را به `client-node1` ببرید، و ریشهٔ Filesystem را روی `/mnt/cephfs-mount-app1` mount کنید. MON اصلی در این لاب `192.168.1.15` (`ceph-node1`) است.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| `/mnt/anisa` / `/mnt/anisa-1` | `/mnt/cephfs-mount-app1` |

## ۱. ساخت کاربر با Capability محدود

اولین تلاش روی `ceph-node1`:

```bash
ceph auth get-or-create client.fs \
  mon 'allow r' \
  osd 'allow rw pool=cephfs_data' \
  mds 'allow r'
```

این کاربر فقط خواندن از MON/MDS و نوشتن در Pool داده دارد. برای لاب آموزشی بعداً آن را با `allow *` عوض کردند.

## ۲. نوشتن keyring در `/etc/ceph`

```bash
cd /etc/ceph
ceph auth get-or-create client.fs \
  mon 'allow *' \
  mds 'allow *' \
  osd 'allow *' \
  -o /etc/ceph/ceph.client.fs.keyring
```

لیست فایل‌ها باید شبیه این باشد:

```text
ceph.cli  ceph.client.admin.keyring  ceph.client.fs.keyring  ceph.conf  rbdmap
```

اگر فایل را اشتباه ساختید:

```bash
rm /etc/ceph/ceph.client.fs.keyring
ceph auth del client.fs
```

و دوباره `get-or-create` را اجرا کنید. در یک لحظه `ceph auth del client.fs` گفت `entity client.fs does not exist` — یعنی کاربر از قبل پاک شده بود؛ همان `get-or-create` کافی است.

نمونهٔ keyring نهایی لاب:

```text
[client.fs]
        key = AQDnonNj/H9ELRAAfZIl28ZnZ5to/10svC0VdQ==
```

## ۳. کپی keyring به کلاینت

از `ceph-node1`:

```bash
scp /etc/ceph/ceph.client.fs.keyring \
  root@client-node1:/etc/ceph/ceph.client.fs.keyring
```

روی `client-node1` محتوا را چک کنید:

```bash
cat /etc/ceph/ceph.client.fs.keyring
```

## ۴. Mount Point

روی `client-node1`:

```bash
mkdir /mnt/cephfs-mount-app1
```

اگر `/mnt/cephfs-mount-app1` از قبل وجود داشت، خطا می‌دهد؛ برای این لاب همان مسیر استفاده شد.

کرنل کلاینت:

```bash
uname -r
```

```text
5.4.0-131-generic
```

## ۵. DNS و `/etc/hosts`

اولین mount با hostname شکست خورد:

```text
server name not found: ceph-node1 (Temporary failure in name resolution)
failed to resolve source
```

روی `client-node1` فایل `/etc/hosts` را کامل کنید. در لاب فقط `client-node1` را داشت و `ceph-node1` را نداشت. حداقل:

```text
192.168.1.15  ceph-node1
192.168.1.10  client-node1
```

تا وقتی DNS درست نشده، از IP مانیتور استفاده کنید.

## ۶. دستور mount

شکل کلی:

```bash
mount -t ceph <mon>:<port>:/ /mnt/cephfs-mount-app1 \
  -o name=<cephx-id>,secret=<key>
```

تلاش‌های لاب و نتیجه:

| دستور | نتیجه |
|---|---|
| `name=fs` + hostname `ceph-node1:6789` بدون hosts | `server name not found` |
| همان + hosts ناقص / MDS laggy | `no mds server is up or the cluster is laggy` |
| `name=cephfs,secret=` (خالی) | `mount option secret requires a value` / `failed to parse ceph options: -22` |
| `name=cephfs` + کلید درست + IP | `mount error 13: Permission denied` |
| `name=fs` + کلید درست + `192.168.1.15:6789` | موفق |

دستور موفق:

```bash
mount -t ceph 192.168.1.15:6789:/ /mnt/cephfs-mount-app1 \
  -o name=fs,secret=AQDnonNj/H9ELRAAfZIl28ZnZ5to/10svC0VdQ==
```

`df -h` باید چیزی شبیه این نشان بدهد:

```text
192.168.1.15:6789:/   54G    0   54G   0% /mnt/cephfs-mount-app1
```

> **توجه:** `name=` برابر است با بخش بعد از `client.` در keyring. اینجا `client.fs` → `name=fs`. نام Filesystem (`cephfs`) را اینجا نگذارید.

> **توجه:** اگر MDS هنوز `laggy or crashed` است، حتی با IP هم mount نمی‌شود. اول `ceph mds stat` را روی کلاستر چک کنید.

## ۷. وضعیت سالم MDS

بعد از برگشتن MDS:

```bash
ceph fs ls
ceph mds stat
```

```text
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
cephfs:1 {0=ceph-node2=up:active}
```

بدون `(laggy or crashed)`.

## خطاهایی که در لاب دیده شد

**Broken pipe هنگام `ceph auth list | grep`**

```text
BrokenPipeError: [Errno 32] Broken pipe
```

خروجی `ceph auth list` طولانی است؛ بستن لوله با `grep` در پایتون ۳ این traceback را می‌دهد. خود دستور auth مشکلی ندارد.

**Blacklist کلاینت CephFS**

اگر کلاینت شبکهٔ ناپایدار داشته باشد، OSD ممکن است آن را blacklist کند (پیش‌فرض حدود یک ساعت). در چت کلاس به این مطلب اشاره شد:

```text
If cephfs client is in a slow or unreliable network environment,
the client will be added to OSD blacklist and OSD map
```

این موضوع در انتهای جلسه با `ceph osd dump` و تلاش برای دستور `blacklist` هم دیده شد؛ دستور لینوکسی `blacklist` وجود ندارد — عملیات روی OSD map با `ceph osd blacklist` / `ceph osd blocklist` است.
</div>
