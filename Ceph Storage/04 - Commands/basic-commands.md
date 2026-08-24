# دستورات پایه Ceph (مرجع سریع)

> بر اساس کلاستر **Ceph Octopus (v15.2.17)** که با **cephadm** دیپلوی شده و شامل سه نود `ceph-node1`، `ceph-node2` و `ceph-node3` است.

---

## 1. وضعیت کلی کلاستر

### `ceph -s` یا `ceph status`

وضعیت کلی سلامت کلاستر، تعداد `mon`، `mgr` و `osd` و همچنین وضعیت PGها را نشان می‌دهد.

این اولین دستوری است که هنگام بررسی وضعیت کلاستر باید اجرا شود.

```bash
ceph -s
# یا
ceph status
```

نمونه خروجی:

```text
cluster:
  id:     733aeaaf-9b58-11f1-b1dc-080027888e11
  health: HEALTH_OK

services:
  mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3 (age 2h)
  mgr: ceph-node1.gyvxkl(active), standbys: ceph-node2.triemz, ceph-node3.mywqip
  osd: 9 osds: 9 up, 9 in

data:
  pools:   1 pools, 1 pgs
  objects: 0 objects, 0 B
  usage:   9.5 GiB used, 183 GiB / 192 GiB avail
  pgs:     1 active+clean
```

### `ceph health detail`

در صورت وجود مشکل، جزئیات کامل هشدارهای سلامت کلاستر را نمایش می‌دهد.

```bash
ceph health detail
```

نمونه:

```text
HEALTH_WARN OSD count 0 < osd_pool_default_size 3

[WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
```

### `ceph -w`

وضعیت کلاستر را به‌صورت زنده نمایش می‌دهد.

برای مشاهده اتفاقاتی مانند `rebalance`، اضافه‌شدن OSD، تغییر وضعیت PGها و سایر رویدادهای لحظه‌ای مفید است.

```bash
ceph -w
```

برای خروج از حالت watch:

```text
Ctrl+C
```

---

## 2. مدیریت میزبان‌ها (Hosts)

### `ceph orch host ls`

لیست تمام نودهایی که تحت مدیریت `cephadm` هستند.

```bash
ceph orch host ls
```

نمونه:

```text
HOST        ADDR             LABELS  STATUS

ceph-node1  ceph-node1
ceph-node2  192.168.111.102
ceph-node3  192.168.111.103
```

### `ceph orch host add <hostname> <ip>`

اضافه‌کردن یک نود جدید به کلاستر.

```bash
ceph orch host add <hostname> <ip>
```

نمونه:

```bash
ceph orch host add ceph-node2 192.168.111.102
```

> این عملیات نیازمند دسترسی SSH مناسب و آماده‌بودن نود مقصد برای مدیریت توسط `cephadm` است.

نمونه خروجی:

```text
Added host 'ceph-node2'
```

### `ceph cephadm check-host <hostname>`

بررسی می‌کند که آیا نود مشخص‌شده برای مدیریت توسط `cephadm` آماده است یا خیر.

مواردی مانند وجود `docker/podman`، `systemctl`، hostname و پیش‌نیازهای سیستم بررسی می‌شوند.

```bash
ceph cephadm check-host ceph-node2
```

نمونه خروجی:

```text
podman|docker (/usr/bin/docker) is present

Hostname "ceph-node2" matches what is expected.

Host looks OK
```

---

## 3. مدیریت دیمون‌ها و سرویس‌ها

### `ceph orch ps`

لیست تمام دیمون‌های در حال اجرا مانند `mon`، `mgr`، `osd`، `grafana` و `prometheus` را نمایش می‌دهد.

```bash
ceph orch ps
```

نمونه:

```text
NAME                     HOST        STATUS         VERSION

mon.ceph-node1           ceph-node1  running (2h)   15.2.17
mgr.ceph-node1.gyvxkl    ceph-node1  running (2h)   15.2.17
osd.0                    ceph-node1  running (1h)   15.2.17
```

### `ceph orch ls`

لیست سرویس‌ها را نمایش می‌دهد؛ نه تک‌تک دیمون‌ها.

همچنین تعداد سرویس‌های در حال اجرا را نسبت به تعداد مورد انتظار نشان می‌دهد.

```bash
ceph orch ls
```

نمونه:

```text
NAME           RUNNING  REFRESHED  AGE  PLACEMENT

mon            3/3      1m ago     2h   ceph-node1,ceph-node2,ceph-node3
mgr            3/3      1m ago     2h   count:3
osd            9/9      1m ago     1h   *
```

### `ceph orch apply mon ceph-node1,ceph-node2,ceph-node3`

مشخص می‌کند که سرویس `mon` روی کدام نودها دیپلوی شود.

```bash
ceph orch apply mon ceph-node1,ceph-node2,ceph-node3
```

نمونه خروجی:

```text
Scheduled mon update...
```

> برای `MON` معمولاً تعداد فرد مانند 3 یا 5 انتخاب می‌شود تا quorum و الگوریتم اجماع به‌درستی کار کنند.

### `ceph orch redeploy <service>`

یک سرویس را مجدداً دیپلوی می‌کند.

این دستور زمانی مفید است که کانفیگ تغییر کرده باشد یا بخواهید یک سرویس مشخص با کانفیگ فعلی مجدداً ایجاد شود.

```bash
ceph orch redeploy <service>
```

نمونه:

```bash
ceph orch redeploy prometheus.ceph-node1
```

نمونه خروجی:

```text
Scheduled to redeploy prometheus.ceph-node1 on host 'ceph-node1'
```

---

## 4. مدیریت OSD و دیسک‌ها

### `ceph orch device ls`

دیسک‌های خام و استفاده‌نشده‌ی هر نود را نمایش می‌دهد؛ دیسک‌هایی که می‌توانند برای ساخت OSD استفاده شوند.

```bash
ceph orch device ls
```

نمونه:

```text
Hostname    Path      Type  Size   Available

ceph-node1  /dev/sdb  hdd   21.4G  Yes
ceph-node1  /dev/sdc  hdd   21.4G  Yes
```

### `ceph orch apply osd --all-available-devices`

تمام دیسک‌های قابل استفاده را به‌صورت خودکار برای OSD استفاده می‌کند.

```bash
ceph orch apply osd --all-available-devices
```

نمونه خروجی:

```text
Scheduled osd.all-available-devices update...
```

> **احتیاط:** این دستور را بدون بررسی `ceph orch device ls` اجرا نکنید. دیسک‌های مناسب می‌توانند به‌صورت خودکار برای OSD استفاده شوند.

### `ceph osd tree`

ساختار سلسله‌مراتبی CRUSH را نمایش می‌دهد.

در این ساختار معمولاً هر هاست یک `bucket` است و OSDهای مربوط به آن زیر هاست قرار می‌گیرند.

```bash
ceph osd tree
```

نمونه:

```text
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT

-1         0.18732  root default

-3         0.06244      host ceph-node1

 0    hdd  0.02081          osd.0            up   1.00000
 1    hdd  0.02081          osd.1            up   1.00000
```

### `ceph osd df`

میزان فضای استفاده‌شده و آزاد را به تفکیک هر OSD نمایش می‌دهد.

```bash
ceph osd df
```

نمونه:

```text
ID  CLASS  WEIGHT  REWEIGHT  SIZE   USE   AVAIL  %USE

 0    hdd  0.02081   1.00000  21GiB  1.1GiB  20GiB  5.2
```

---

## 5. فضای ذخیره‌سازی و Poolها

### `ceph df`

فضای کلی کلاستر و میزان مصرف آن را نمایش می‌دهد و اطلاعات Poolها را نیز ارائه می‌کند.

```bash
ceph df
```

نمونه:

```text
--- RAW STORAGE ---

CLASS  SIZE    AVAIL   USED    RAW USED  %RAW USED

hdd    192GiB  183GiB  9.5GiB     9.5GiB       4.95
```

### `ceph osd pool ls`

لیست Poolهای موجود در کلاستر را نمایش می‌دهد.

```bash
ceph osd pool ls
```

نمونه:

```text
device_health_metrics
```

### `ceph pg stat`

خلاصه وضعیت `Placement Group`ها را نمایش می‌دهد.

برای بررسی وضعیت‌هایی مانند `active+clean`، `degraded`، `recovering` و `rebalance` کاربرد دارد.

```bash
ceph pg stat
```

نمونه:

```text
1 pgs: 1 active+clean; 0 B data, 9.5 GiB used, 183 GiB / 192 GiB avail
```

### `ceph osd stat`

یک خلاصه بسیار کوتاه از تعداد OSDها و وضعیت `up/in` ارائه می‌کند.

برای چک سریع وضعیت OSDها مناسب است.

```bash
ceph osd stat
```

نمونه:

```text
9 osds: 9 up (since 10m), 9 in (since 2h); epoch: e45
```

### `ceph osd stats`

مشابه `ceph osd stat` است، اما اطلاعات بیشتری را در قالب JSON ارائه می‌دهد.

بیشتر برای اسکریپت‌ها و سیستم‌های مانیتورینگ کاربرد دارد.

```bash
ceph osd stats
```

نمونه:

```json
{
  "epoch": 45,
  "num_osds": 9,
  "num_up_osds": 9,
  "num_in_osds": 9,
  "num_remapped_pgs": 0
}
```

### `ceph osd pool stats`

آمار I/O و throughput را به تفکیک Pool نمایش می‌دهد.

اطلاعاتی مانند read/write bytes per second و تعداد عملیات را می‌توان از خروجی بررسی کرد.

```bash
ceph osd pool stats
```

نمونه:

```text
pool device_health_metrics id 1
  nothing is going on
```

---

## 6. نسخه و اطلاعات کلی

### `ceph -v`

نسخه ابزار CLI نصب‌شده روی هاست فعلی را نمایش می‌دهد.

```bash
ceph -v
```

نمونه:

```text
ceph version 15.2.17 (cc0d41808940bac556de2027dfddd68d3b6558f9) octopus (stable)
```

### `ceph versions`

نسخه دیمون‌های مختلف در کل کلاستر را نمایش می‌دهد.

برای بررسی یکسان‌بودن نسخه Ceph روی تمام نودها بسیار مفید است.

```bash
ceph versions
```

نمونه:

```json
{
  "mon": {
    "ceph version 15.2.17 (...) octopus (stable)": 3
  },
  "mgr": {
    "ceph version 15.2.17 (...) octopus (stable)": 3
  },
  "osd": {
    "ceph version 15.2.17 (...) octopus (stable)": 9
  }
}
```

### `ceph mon stat`

وضعیت خلاصه Monitorها و quorum فعلی را نمایش می‌دهد.

```bash
ceph mon stat
```

نمونه:

```text
e3: 3 mons at {ceph-node1=...,ceph-node2=...,ceph-node3=...},
election epoch 12,
leader 0 ceph-node1,
quorum 0,1,2
```

---

## 7. دسترسی به شل کانتینری Ceph

### `cephadm shell`

یک shell موقت داخل کانتینر Ceph ایجاد می‌کند و امکان استفاده از دستورات Ceph را فراهم می‌کند.

این روش زمانی بسیار مفید است که ابزار `ceph-common` روی هاست نصب نشده باشد.

```bash
cephadm shell
```

در صورت نیاز به مشخص‌کردن `FSID`، فایل کانفیگ و keyring:

```bash
sudo cephadm shell \
  --fsid <fsid> \
  -c /etc/ceph/ceph.conf \
  -k /etc/ceph/ceph.client.admin.keyring
```

> در محیط‌های `cephadm`، قبل از نصب مستقیم پکیج‌های Ceph روی هاست، استفاده از `cephadm shell` را در نظر بگیرید؛ این روش محیط CLI را با کانتینرهای همان کلاستر هم‌راستا نگه می‌دارد.
