# دستورات پایه Ceph (مرجع سریع)

> بر اساس کلاستر Octopus (v15.2.17) دیپلوی‌شده با cephadm — سه نود (ceph-node1, ceph-node2, ceph-node3)

---

## 1. وضعیت کلی کلاستر

### `ceph -s` یا `ceph status`
وضعیت کلی سلامت کلاستر، تعداد mon/mgr/osd، و وضعیت PGها را نشان می‌دهد. اولین دستوری که باید هر بار چک کنید.

```
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
جزئیات کامل هشدارها را در صورت وجود مشکل نشان می‌دهد (مثلاً OSD خراب یا کمبود مانیتور).

```
HEALTH_WARN OSD count 0 < osd_pool_default_size 3
[WRN] TOO_FEW_OSDS: OSD count 0 < osd_pool_default_size 3
```

### `ceph -w`
همان `ceph -s` است ولی به‌صورت زنده (watch) — برای دیدن لحظه‌ای پروسه‌هایی مثل rebalance یا اضافه‌شدن OSD مفید است.

---

## 2. مدیریت میزبان‌ها (Hosts)

### `ceph orch host ls`
لیست همه نودهایی که تحت مدیریت cephadm هستند.

```
HOST        ADDR             LABELS  STATUS
ceph-node1  ceph-node1
ceph-node2  192.168.111.102
ceph-node3  192.168.111.103
```

### `ceph orch host add <hostname> <ip>`
اضافه‌کردن نود جدید به کلاستر (نیاز به SSH key کلاستر روی نود مقصد دارد).

```
Added host 'ceph-node2'
```

### `ceph cephadm check-host <hostname>`
بررسی می‌کند نود مشخص‌شده برای مدیریت توسط cephadm آماده است یا نه (داکر، systemctl، هاست‌نیم و...).

```
podman|docker (/usr/bin/docker) is present
Hostname "ceph-node2" matches what is expected.
Host looks OK
```

---

## 3. مدیریت دیمون‌ها و سرویس‌ها

### `ceph orch ps`
لیست تمام دیمون‌های در حال اجرا (mon, mgr, osd, grafana و...) به همراه هاست، وضعیت و ورژن.

```
NAME                     HOST        STATUS         VERSION
mon.ceph-node1           ceph-node1  running (2h)   15.2.17
mgr.ceph-node1.gyvxkl    ceph-node1  running (2h)   15.2.17
osd.0                    ceph-node1  running (1h)   15.2.17
```

### `ceph orch ls`
لیست سرویس‌ها (نه تک‌تک دیمون‌ها) به همراه تعداد در حال اجرا نسبت به هدف.

```
NAME           RUNNING  REFRESHED  AGE  PLACEMENT
mon            3/3      1m ago     2h   ceph-node1,ceph-node2,ceph-node3
mgr            3/3      1m ago     2h   count:3
osd            9/9      1m ago     1h   *
```

### `ceph orch apply mon ceph-node1,ceph-node2,ceph-node3`
مشخص‌کردن روی کدام نودها mon دیپلوی شود (تعداد فرد به‌خاطر الگوریتم Paxos).

```
Scheduled mon update...
```

### `ceph orch redeploy <service>`
ری‌دیپلوی یک سرویس با کانفیگ به‌روز (مثلاً بعد از تغییر هاست‌نیم یا رفع مشکل کانفیگ).

```
Scheduled to redeploy prometheus.ceph-node1 on host 'ceph-node1'
```

---

## 4. مدیریت OSD و دیسک‌ها

### `ceph orch device ls`
لیست دیسک‌های خام و استفاده‌نشده‌ی هر نود که پتانسیل تبدیل‌شدن به OSD دارند.

```
Hostname    Path      Type  Size   Available
ceph-node1  /dev/sdb  hdd   21.4G  Yes
ceph-node1  /dev/sdc  hdd   21.4G  Yes
```

### `ceph orch apply osd --all-available-devices`
تمام دیسک‌های آزاد روی همه‌ی نودها را به‌طور خودکار به‌عنوان OSD کلیم می‌کند.

```
Scheduled osd.all-available-devices update...
```

### `ceph osd tree`
نمایش سلسله‌مراتب CRUSH — هر هاست به‌عنوان یک bucket و OSDهای زیرمجموعه‌اش.

```
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT
-1         0.18732  root default
-3         0.06244      host ceph-node1
 0    hdd  0.02081          osd.0            up   1.00000
 1    hdd  0.02081          osd.1            up   1.00000
```

### `ceph osd df`
میزان فضای استفاده‌شده/آزاد به تفکیک هر OSD.

```
ID  CLASS  WEIGHT  REWEIGHT  SIZE   USE   AVAIL  %USE
 0    hdd  0.02081   1.00000  21GiB  1.1GiB  20GiB  5.2
```

---

## 5. فضای ذخیره‌سازی و Poolها

### `ceph df`
میزان فضای کلی کلاستر و استفاده‌شده به تفکیک هر pool.

```
--- RAW STORAGE ---
CLASS  SIZE    AVAIL   USED    RAW USED  %RAW USED
hdd    192GiB  183GiB  9.5GiB     9.5GiB       4.95
```

### `ceph osd pool ls`
لیست تمام poolهای موجود در کلاستر.

```
device_health_metrics
```

### `ceph pg stat`
خلاصه‌ی وضعیت Placement Groupها (فعال، تمیز، در حال ری‌بالانس و...).

```
1 pgs: 1 active+clean; 0 B data, 9.5 GiB used, 183 GiB / 192 GiB avail
```

### `ceph osd stat`
خلاصه‌ی خیلی کوتاه از تعداد OSDها و وضعیت up/in — یک‌خطی، برای چک سریع.

```
9 osds: 9 up (since 10m), 9 in (since 2h); epoch: e45
```

### `ceph osd stats`
مشابه `osd stat` ولی خروجی JSON با جزئیات بیشتر (epoch، تعداد osdmap و...) — بیشتر برای اسکریپت/مانیتورینگ خودکار کاربرد دارد تا خواندن دستی.

```
{"epoch": 45, "num_osds": 9, "num_up_osds": 9, "num_in_osds": 9, "num_remapped_pgs": 0}
```

### `ceph osd pool stats`
آمار I/O و throughput هر pool به‌صورت جداگانه (read/write bytes per second، تعداد عملیات و...).

```
pool device_health_metrics id 1
  nothing is going on
```

---

## 6. نسخه و اطلاعات کلی

### `ceph -v`
نسخه‌ی نصب‌شده‌ی خود ابزار CLI روی هاست فعلی.

```
ceph version 15.2.17 (cc0d41808940bac556de2027dfddd68d3b6558f9) octopus (stable)
```

### `ceph versions`
نسخه‌ی هر دیمون در کل کلاستر — برای اطمینان از یکسان‌بودن نسخه‌ها روی همه نودها مفید است.

```
{
    "mon": {"ceph version 15.2.17 (...) octopus (stable)": 3},
    "mgr": {"ceph version 15.2.17 (...) octopus (stable)": 3},
    "osd": {"ceph version 15.2.17 (...) octopus (stable)": 9}
}
```

### `ceph mon stat`
وضعیت خلاصه‌ی مانیتورها و کوروم فعلی.

```
e3: 3 mons at {ceph-node1=...,ceph-node2=...,ceph-node3=...}, election epoch 12, leader 0 ceph-node1, quorum 0,1,2
```

---

## 7. دسترسی به شل کانتینری Ceph (بدون نصب ceph-common)

### `cephadm shell`
وارد یک کانتینر موقت با دسترسی کامل به دستورات `ceph` می‌شود — وقتی `ceph-common` روی هاست نصب نیست کاربردی است.

```
sudo cephadm shell --fsid <fsid> -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring
```
