<div dir="rtl" lang="fa">

# فعال کردن Dashboard، کاربر admin، و وصل Grafana

روی `ceph-node1` ریپوی APT را از Octopus به Pacific بردند، پکیج `ceph-mgr-dashboard` را بالا آوردند، بعد UI روی `https://192.168.1.15:8443` باز شد. کاربر وب Dashboard با کاربر CephX فرق دارد.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| pool `anisa` | `rbd-pool-app1` |
| `client.shobeyr` | `client.app1` |

## ۱. ریپو: اسم فایل Octopus، محتوای Pacific

روی `ceph-node2` فایل هنوز اسم قدیمی داشت:

```bash
cd /etc/apt/sources.list.d/
ls
nano download_ceph_com_debian_octopus.list
```

محتوا را به Pacific عوض کردند:

```text
deb http://download.ceph.com/debian-pacific focal main
```

بعد `apt upgrade` پکیج‌ها را از `15.2.17-1focal` (Octopus) به `16.2.10-1focal` (Pacific) برد، از جمله:

```text
ceph-mgr-dashboard (16.2.10-1focal) over (15.2.17-1focal)
ceph-mgr-cephadm
ceph-mgr
ceph-mds
ceph-osd
```

`ceph versions` بعد از این کار هنوز مخلوط بود:

```text
"osd":  "ceph version 16.2.10 ... pacific (stable)": 9
"mon":  pacific 16.2.10
"mds":  pacific 16.2.10
"mgr":  pacific 16.2.10  (چند daemon)
        quincy  17.2.5   (حداقل یک MGR)
```

روی خود `ceph-node1` دستور `ceph -v` یک‌بار **Quincy 17.2.5** چاپ کرد. یعنی CLI یا یک MGR از ریپوی `debian-quincy` آمده بود.

> **هشدار:** کلاستر مخلوط Pacific/Quincy برای لاب آموزشی خطرناک است. ارتقا باید روی همهٔ daemonها یک نسخه باشد. اینجا هدف فقط بالا آمدن ماژول Dashboard بود.

## ۲. پورت ۸۴۴۳ اول Connection refused

Dashboard روی HTTPS گوش می‌دهد تا مرورگر به UI مدیریت وصل شود. `create-self-signed-cert` یک گواهی موقت می‌سازد (مرورگر warning می‌دهد؛ برای لاب کافی است). تا گواهی و bind روی `8443` نباشد، `curl http://…:8443` با Connection refused برمی‌گردد — هم به‌خاطر `http` به‌جای `https`، هم چون هنوز هیچ‌چیز به پورت bind نشده.

```bash
docker ps
curl http://192.168.1.15:8443
```

خروجی لاب:

```text
curl: (7) Failed to connect to 192.168.1.15 port 8443: Connection refused
```

مرورگر روی `192.168.1.17` هم `ERR_CONNECTION_REFUSED` داد. دو نکته:

- Dashboard معمولاً **HTTPS** است (`https://…:8443`)، نه `http://`.
- تا MGR ماژول را bind نکند، پورت بسته است.

در چت کلاس این دستورها آمد (استاد بعضی را روی نود زد، بعضی را دانشجوها فرستادند):

```bash
ceph dashboard create-self-signed-cert

ceph config set mgr mgr/dashboard/server_addr 192.168.1.15
ceph config set mgr mgr/dashboard/server_port 8080
ceph config set mgr mgr/dashboard/ssl_server_port 8443
ceph config set mgr mgr/dashboard/ssl false
```

آدرس نهایی که UI باز شد:

```text
https://192.168.1.15:8443
```

لیست Pool در UI همان لاب: `anis` (application `mgr_devicehealth`)، `rbd-pool-app1`، `cephfs_data`، `cephfs_metadata`، `rbd`. صفحهٔ Monitoring/Alerts بدون URL پرومتهوس خالی می‌ماند.

## ۳. کاربر Dashboard (نه CephX)

ورود وب با `ceph dashboard ac-user-create` ساخته می‌شود. این `admin` کاربر CephX به نام `client.admin` نیست: یکی برای مرورگر است، یکی برای CLI روی MON. نقش `administrator` فقط داخل خود UI Dashboard معنی دارد.

رمز را در فایل بگذارید؛ `-i` مسیر فایل می‌خواهد:

```bash
echo "P@ssw0rd" > pass.txt
ceph dashboard ac-user-create admin -i pass.txt administrator
```

در لاب استاد اول `nano password` کرد و بعد:

```bash
ceph dashboard ac-user-create admin -i
```

نقش آخر `administrator` است. نقش‌های دیگر در سند SUSE که در چت لینک شد آمده؛ در لاب فقط administrator ساخته شد.

## ۴. وصل Grafana / Prometheus / Alertmanager

Dashboard نمودار را خودش scrape نمی‌کند؛ فقط iframe/API به Grafana می‌زند. اگر این URLها خالی باشند، صفحهٔ Monitoring/Alerts خالی می‌ماند — Grafana روی ۳۰۰۰ ممکن است سالم باشد ولی Dashboard نمی‌داند کجاست.

Containerها روی همین نود بودند. دستورهای چت:

```bash
ceph dashboard set-grafana-api-url http://192.168.1.15:3000
ceph dashboard set-prometheus-api-host http://192.168.1.15:9090
ceph dashboard set-alertmanager-api-host http://192.168.1.15:9093
```

پورت دقیق را از `docker ps` همان نود بخوانید؛ اسلاید کلاس فقط `'API:PORT'` نوشت.

صفحهٔ Alerts در Dashboard تا این URLها ست نشوند پیام می‌دهد که Prometheus/Alertmanager را وصل کنید.

## ۵. مرور CephX و HAProxy (لاب این جلسه نیست)

آخر جلسه `ceph auth get-or-create client.app1` با Pool `rbd-pool-app1` زده شد — همان الگوی `05 - Cephx`. کاربرهای `client.admin` را از نو نسازید.

جستجوی Google برای **HAProxy + RGW** و اشاره به NFS-Ganesha در چت ماند؛ نصب نشد.

```bash
ceph auth get-or-create client.app1 \
  mon 'allow r' \
  osd 'allow rwx pool=rbd-pool-app1'
```
</div>
