<div dir="rtl" lang="fa">

# RADOS Gateway: Object Storage با S3 و Swift

نیمهٔ دوم جلسهٔ ششم Object Storage است: **RGW** (`radosgw`) یک HTTP API روی RADOS می‌گذارد تا کلاینت‌ها با پروتکل **S3** (آمازون) یا **Swift** (OpenStack) کار کنند.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| `anisa-rgw` | `rgw-user-app1` |
| `anisa-rgw:swift` | `rgw-user-app1:swift` |
| `anisa-bucket` | `rgw-bucket-app1` |

مسیرهای **RBD** و **CephFS** کلاینت را به کلاستر وصل می‌کنند. RGW همان داده را از مسیر REST می‌دهد؛ اپلیکیشن نیازی به کتابخانهٔ `librados` ندارد. یعنی بک‌آپ، سایت استاتیک یا SDK آمازون فقط HTTP می‌زند — لازم نیست پکیج Ceph روی آن ماشین باشد.

دنیای Object سه مفهوم دارد:

| مفهوم | یعنی چه |
|---|---|
| **Object** | یک فایل با شناسه و metadata (مثلاً یک عکس) |
| **Bucket** (S3) / **Container** (Swift) | پوشه‌ای منطقی که objectها داخلش هستند — نه دایرکتوری POSIX |
| **Access key / Secret** | جفت کلید HTTP؛ شبیه username/password برای API، نه کاربر CephX |

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

- پروتکل **S3 API** — لهجهٔ HTTP آمازون؛ رایج‌تر؛ ابزارهایی مثل `s3cmd`، AWS SDK، PHP SDK.
- پروتکل **Swift API** — لهجهٔ HTTP اوپن‌استک برای *همان* داده روی RADOS. در این لاب با CLI به نام `swift` روی `client-node1` تست شد.
- پروتکل **Admin API** — `radosgw-admin` برای ساخت کاربر، subuser، کلید.

پروتکل‌های **S3** و **Swift** دو لهجهٔ جدا هستند، نه دو کلاستر. کاربر Object می‌تواند هر دو را داشته باشد (در لاب uid برای S3 و subuser برای Swift).

سرویس **RGW** یک daemon جدا است (`rgw.rgw-node1.rgw0`)، نه روی MON/OSD. در لاب روی VM جدا به نام `rgw-node1` با IP `192.168.1.11` نصب شد.

## Poolهایی که Ansible می‌سازد

این Poolها فایل‌سیستم POSIX نیستند. RGW ایندکس bucket، metadata کاربر و لاگ را روی RADOS می‌گذارد:

| Pool (نمونه) | نقش تقریبی |
|---|---|
| `.rgw.root` | تنظیمات پایهٔ realm/zone |
| `default.rgw.meta` | کاربر و metadata |
| `default.rgw.buckets.index` | فهرست objectهای داخل bucket |
| `default.rgw.log` | لاگ عملیات |
| `default.rgw.control` | هماهنگی داخلی daemon |

بعد از Playbook این Poolها ظاهر شدند:

```text
.rgw.root
default.rgw.log
default.rgw.control
default.rgw.meta
default.rgw.buckets.index
```

دستور `ceph -s` باید خط سرویس RGW را نشان بدهد:

```text
rgw: 1 daemon active (rgw-node1.rgw0)
```

تعداد Pool از ۶ به حدود ۱۰ می‌رسد و PGها از ۱۹۷ به حدود ۳۰۱.

## Frontend: Civetweb روی پورت ۸۰۸۰

لایهٔ **Frontend** همان HTTP server داخل فرآیند `radosgw` است. **Civetweb** پیش‌فرض قدیمی‌تر (همین دوره / Octopus)؛ **Beast** جایگزین جدیدتر روی نسخه‌های بعدی. پورت `8080` یعنی کلاینت به `http://rgw-node1:8080` می‌زند، نه به MON روی `6789`.

در `ceph-ansible/group_vars/all.yml`:

```yaml
# radosgw_frontend_type: beast
radosgw_civetweb_port: 8080
```

نسخهٔ **Octopus** در این دوره هنوز Civetweb را به‌عنوان frontend پیش‌فرض دارد. Endpoint لاب:

```text
http://192.168.1.11:8080
```

مسیر Auth Swift نسخهٔ ۱:

```text
http://192.168.1.11:8080/auth/1.0
```

## کاربر RGW در برابر کاربر CephX

دو لایه کلید وجود دارد و قاطی کردنشان رایج‌ترین اشتباه است:

| لایه | نمونه در لاب | کار |
|---|---|---|
| CephX برای خود daemon | `client.rgw.rgw-node1.rgw0` در `/var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring` | تا RGW به MON/OSD وصل شود — مثل کلید هر daemon دیگر |
| کاربر Object (S3/Swift) | `uid=rgw-user-app1` و subuser `rgw-user-app1:swift` | تا کلاینت HTTP به bucket برسد — این کلید داخل CephX نیست |

دستور `radosgw-admin` را باید با keyring خود RGW صدا بزنید (`-k` و `--name`)، نه با `client.admin` روی نودی که keyring RGW ندارد. دلیل: دستور ادمین از خودِ Gateway می‌پرسد، نه از MON؛ پس باید با هویت daemon RGW احراز شود.

## Keystone (فقط بحث کلاس)

اگر RGW پشت OpenStack باشد، می‌توان Auth را به Keystone سپرد. در چت کلاس نمونه‌ای از تنظیمات mimic/latest آمد؛ در لاب عملی Keystone بالا نیامد.

نمونهٔ گزینه‌ها:

```text
rgw keystone url = http://192.168.1.111:5000
rgw keystone accepted roles = admin, Member, swiftoperator
rgw s3 auth use keystone = true
```

مستندات اشاره‌شده در جلسه:

- آدرس `https://docs.ceph.com/en/latest/radosgw/keystone/`
- مثال‌های PHP S3 در docs.ceph.com

## نظارت

انتهای جلسه مانیتورینگ جدا از RGW بود (Grafana، Prometheus، Zabbix در چت). خود RGW را از `ceph -s` و Poolهای `.rgw.*` دنبال کنید.
</div>
