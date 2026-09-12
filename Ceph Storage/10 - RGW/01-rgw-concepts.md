<div dir="rtl" lang="fa">

# RADOS Gateway: Object Storage با S3 و Swift

نیمهٔ دوم جلسهٔ ششم Object Storage است: **RGW** (`radosgw`) یک HTTP API روی RADOS می‌گذارد تا کلاینت‌ها با پروتکل **S3** (آمازون) یا **Swift** (OpenStack) کار کنند.

RBD و CephFS کلاینت را به کلاستر وصل می‌کنند. RGW همان داده را از مسیر REST می‌دهد؛ اپلیکیشن نیازی به کتابخانهٔ `librados` ندارد.

## معماری

اسلاید جلسه:

```text
CLIENTS
  YOUR APP ──► Direct Access ──► Ceph Cluster
  Swift API ─┐
  S3 API    ─┼──► Rados Gateway ──► Ceph Cluster
  Admin API ─┘
              RESTful HTTP / S Access
```

- **S3 API** — رایج‌تر؛ ابزارهایی مثل `s3cmd`، AWS SDK، PHP SDK.
- **Swift API** — در این لاب با CLI به نام `swift` روی `client-node1` تست شد.
- **Admin API** — `radosgw-admin` برای ساخت کاربر، subuser، کلید.

RGW یک daemon جدا است (`rgw.rgw-node1.rgw0`)، نه روی MON/OSD. در لاب روی VM جدا به نام `rgw-node1` با IP `192.168.1.11` نصب شد.

## Poolهایی که Ansible می‌سازد

بعد از Playbook این Poolها ظاهر شدند:

```text
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
default.rgw.buckets.index
```

`ceph -s` باید خط سرویس RGW را نشان بدهد:

```text
rgw: 1 daemon active (rgw-node1.rgw0)
```

تعداد Pool از ۶ به حدود ۱۰ می‌رسد و PGها از ۱۹۷ به حدود ۳۰۱.

## Frontend: Civetweb روی پورت ۸۰۸۰

در `ceph-ansible/group_vars/all.yml`:

```yaml
# radosgw_frontend_type: beast
radosgw_civetweb_port: 8080
```

Octopus در این دوره هنوز Civetweb را به‌عنوان frontend پیش‌فرض دارد. Endpoint لاب:

```text
http://192.168.1.11:8080
```

Auth Swift نسخهٔ ۱:

```text
http://192.168.1.11:8080/auth/1.0
```

## کاربر RGW در برابر کاربر CephX

دو لایه کلید وجود دارد:

| لایه | نمونه در لاب | کار |
|---|---|---|
| CephX برای خود daemon | `client.rgw.rgw-node1.rgw0` در `/var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring` | تا RGW به MON/OSD وصل شود |
| کاربر Object (S3/Swift) | `uid=anisa-rgw` و subuser `anisa-rgw:swift` | تا کلاینت HTTP به bucket برسد |

`radosgw-admin` را باید با keyring خود RGW صدا بزنید (`-k` و `--name`)، نه با `client.admin` روی نودی که keyring RGW ندارد.

## Keystone (فقط بحث کلاس)

اگر RGW پشت OpenStack باشد، می‌توان Auth را به Keystone سپرد. در چت کلاس نمونه‌ای از تنظیمات mimic/latest آمد؛ در لاب عملی Keystone بالا نیامد.

نمونهٔ گزینه‌ها:

```text
rgw keystone url = http://192.168.1.111:5000
rgw keystone accepted roles = admin, Member, swiftoperator
rgw s3 auth use keystone = true
```

مستندات اشاره‌شده در جلسه:

- `https://docs.ceph.com/en/latest/radosgw/keystone/`
- PHP S3 examples در docs.ceph.com

## نظارت

انتهای جلسه مانیتورینگ جدا از RGW بود (Grafana، Prometheus، Zabbix در چت). خود RGW را از `ceph -s` و Poolهای `.rgw.*` دنبال کنید.
</div>
