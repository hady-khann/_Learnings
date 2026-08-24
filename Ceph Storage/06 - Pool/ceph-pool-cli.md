# مدیریت Ceph Pool با دستورات CLI

این فایل روش مدیریت مستقیم Poolهای Ceph را با استفاده از دستورات `ceph` توضیح می‌دهد.

در این روش از Ansible استفاده نمی‌شود و تمام عملیات مستقیماً روی Cluster Ceph انجام می‌شوند.

---

# 1. بررسی وضعیت Cluster

قبل از هرگونه تغییر در Poolها، ابتدا وضعیت Cluster را بررسی کنید:

```bash
ceph -s
```

یا:

```bash
ceph status
```

همچنین می‌توانید وضعیت دقیق‌تر OSDها را بررسی کنید:

```bash
ceph osd stat
```

و:

```bash
ceph osd tree
```

> قبل از ایجاد یا تغییر Pool، مطمئن شوید Cluster وضعیت قابل‌قبولی دارد و OSDهای موردنیاز در دسترس هستند.

---

# 2. مشاهده Poolهای موجود

برای مشاهده نام Poolها:

```bash
ceph osd pool ls
```

برای مشاهده جزئیات:

```bash
ceph osd pool ls detail
```

این دستور اطلاعاتی مانند موارد زیر را نشان می‌دهد:

```text
pool name
pool id
pool size
min_size
pg_num
pgp_num
crush_rule
application
autoscale mode
```

برای مشاهده یک Pool مشخص:

```bash
ceph osd pool get mypool all
```

مثال:

```bash
ceph osd pool get rbd all
```

---

# 3. ایجاد Replicated Pool

برای ایجاد یک Pool معمولی از نوع `replicated`:

```bash
ceph osd pool create mypool 32
```

در این دستور:

```text
mypool → نام Pool
32     → pg_num
```

پس از ایجاد Pool، می‌توانید تنظیمات آن را بررسی کنید:

```bash
ceph osd pool get mypool all
```

---

# 4. تعیین Replica Size

برای تعیین تعداد Replicaها:

```bash
ceph osd pool set mypool size 3
```

یعنی هر Object به‌صورت 3 Replica روی OSDهای مختلف نگهداری می‌شود.

بررسی:

```bash
ceph osd pool get mypool size
```

خروجی مورد انتظار:

```text
size: 3
```

---

# 5. تعیین `min_size`

برای تعیین حداقل Replicaهای موردنیاز برای سرویس‌دهی I/O:

```bash
ceph osd pool set mypool min_size 2
```

بررسی:

```bash
ceph osd pool get mypool min_size
```

یک تنظیم رایج در Pool با `size=3`:

```text
size     = 3
min_size = 2
```

یعنی در شرایط Failure، Pool می‌تواند با حداقل 2 Replica سالم همچنان I/O را سرویس دهد.

> مقدار مناسب `min_size` به طراحی Failure Domain و سیاست تحمل خرابی Cluster بستگی دارد. این مقدار را صرفاً کپی نکنید.

---

# 6. تعیین CRUSH Rule

برای مشاهده CRUSH Ruleهای موجود:

```bash
ceph osd crush rule ls
```

برای مشاهده جزئیات Ruleها:

```bash
ceph osd crush rule dump
```

برای تعیین Rule یک Pool:

```bash
ceph osd pool set mypool crush_rule replicated_rule
```

بررسی:

```bash
ceph osd pool get mypool crush_rule
```

مثال:

```text
crush_rule: replicated_rule
```

> CRUSH Rule مشخص می‌کند Replicaهای Pool چگونه در Storage hierarchy توزیع شوند. در Production، انتخاب Rule باید با Failure Domain واقعی Cluster هماهنگ باشد.

---

# 7. فعال‌کردن Application روی Pool

Poolهای Ceph معمولاً باید Application مناسب خود را داشته باشند.

## RBD

برای Pool مربوط به Block Storage:

```bash
ceph osd pool application enable mypool rbd
```

بررسی:

```bash
ceph osd pool application get mypool
```

---

## CephFS

برای Pool مورد استفاده CephFS:

```bash
ceph osd pool application enable mypool cephfs
```

بررسی:

```bash
ceph osd pool application get mypool
```

> در یک CephFS واقعی معمولاً Poolها توسط فرآیند راه‌اندازی CephFS مدیریت می‌شوند؛ بنابراین قبل از تغییر دستی Application، ساختار CephFS را بررسی کنید.

---

## RGW

برای Object Storage:

```bash
ceph osd pool application enable mypool rgw
```

بررسی:

```bash
ceph osd pool application get mypool
```

---

# 8. فعال‌کردن PG Autoscaler

برای فعال‌کردن Autoscaler:

```bash
ceph osd pool set mypool pg_autoscale_mode on
```

بررسی:

```bash
ceph osd pool get mypool pg_autoscale_mode
```

خروجی:

```text
pg_autoscale_mode: on
```

---

# 9. حالت‌های PG Autoscaler

سه حالت اصلی وجود دارد.

## `on`

Ceph اجازه دارد تعداد PGها را به‌صورت خودکار مدیریت کند:

```bash
ceph osd pool set mypool pg_autoscale_mode on
```

---

## `warn`

Ceph فقط هشدار می‌دهد:

```bash
ceph osd pool set mypool pg_autoscale_mode warn
```

در این حالت PGها به‌صورت خودکار تغییر نمی‌کنند.

---

## `off`

Autoscaler غیرفعال می‌شود:

```bash
ceph osd pool set mypool pg_autoscale_mode off
```

در این حالت مدیریت PG کاملاً دستی است.

---

# 10. مشاهده وضعیت PG Autoscaler

مهم‌ترین دستور برای بررسی Autoscaler:

```bash
ceph osd pool autoscale-status
```

این دستور وضعیت Poolها و تصمیم Autoscaler را نشان می‌دهد.

برای مثال:

```text
POOL        SIZE   TARGET SIZE   RATE   RATIO   TARGET RATIO   PG_NUM   NEW PG_NUM   AUTOSCALE
mypool      ...    ...           ...    ...     0.20           32       ...          on
```

ستون‌های مهم:

```text
SIZE
RATIO
TARGET RATIO
PG_NUM
NEW PG_NUM
AUTOSCALE
```

---

# 11. تنظیم `target_size_ratio`

اگر انتظار دارید Pool تقریباً 20٪ از ظرفیت Cluster را مصرف کند:

```bash
ceph osd pool set mypool target_size_ratio 0.2
```

بررسی:

```bash
ceph osd pool get mypool target_size_ratio
```

مثال:

```text
0.2 = 20%
```

مقادیر نمونه:

```text
0.1 = 10%
0.2 = 20%
0.5 = 50%
1.0 = 100%
```

این مقدار یک **Hint برای PG Autoscaler** است و محدودیت سخت برای اندازه Pool محسوب نمی‌شود.

---

# 12. تنظیم دستی `pg_num`

اگر Autoscaler خاموش باشد:

```bash
ceph osd pool set mypool pg_num 32
```

بررسی:

```bash
ceph osd pool get mypool pg_num
```

> تغییر `pg_num` می‌تواند باعث PG splitting و rebalancing شود. در Cluster واقعی، قبل از تغییر دستی PGها تأثیر آن را بررسی کنید.

---

# 13. تنظیم `pgp_num`

در تنظیمات معمول:

```bash
ceph osd pool set mypool pgp_num 32
```

بررسی:

```bash
ceph osd pool get mypool pgp_num
```

در حالت عادی:

```text
pg_num  = pgp_num
```

است.

> در Clusterهای مدرن معمولاً نباید بدون دلیل این دو مقدار را متفاوت تنظیم کنید.

---

# 14. ایجاد Erasure-Coded Pool

ابتدا Profileهای موجود را مشاهده کنید:

```bash
ceph osd erasure-code-profile ls
```

برای مشاهده Profile:

```bash
ceph osd erasure-code-profile get default
```

سپس Pool را ایجاد کنید:

```bash
ceph osd pool create archive-pool 32 erasure default
```

ساختار کلی:

```text
ceph osd pool create <pool> <pg_num> erasure <profile>
```

مثال:

```bash
ceph osd pool create archive-pool 32 erasure default
```

بعد Application مناسب را فعال کنید:

```bash
ceph osd pool application enable archive-pool rgw
```

بررسی:

```bash
ceph osd pool ls detail
```

> Erasure Coding برای همه Workloadها مناسب نیست. برای RBD یا Workloadهای Latency-sensitive، قبل از استفاده باید محدودیت‌های نسخه Ceph و Application موردنظر بررسی شوند.

---

# 15. مشاهده تنظیمات کامل یک Pool

برای دیدن تمام تنظیمات:

```bash
ceph osd pool get mypool all
```

این یکی از مهم‌ترین دستورات Troubleshooting است.

برای مثال:

```bash
ceph osd pool get mypool all
```

می‌تواند اطلاعاتی مانند:

```text
size
min_size
pg_num
pgp_num
crush_rule
hashpspool
target_size_ratio
pg_autoscale_mode
application
```

را نشان دهد.

---

# 16. مشاهده Applicationهای Pool

```bash
ceph osd pool application get mypool
```

مثال:

```text
{
    "rbd": {}
}
```

یا برای مشاهده Applicationهای تمام Poolها:

```bash
ceph osd pool ls detail
```

---

# 17. مشاهده تنظیمات Replica

```bash
ceph osd pool get mypool size
```

و:

```bash
ceph osd pool get mypool min_size
```

مثلاً:

```text
size: 3
min_size: 2
```

---

# 18. تغییر نام Pool

برای تغییر نام Pool:

```bash
ceph osd pool rename mypool newpool
```

سپس:

```bash
ceph osd pool ls
```

را اجرا کنید تا نام جدید را بررسی کنید.

> اگر Pool توسط Application دیگری مانند RBD، CephFS یا RGW استفاده می‌شود، قبل از Rename وابستگی‌های Application را بررسی کنید.

---

# 19. حذف Pool

برای حذف Pool، ابتدا Pool را بررسی کنید:

```bash
ceph osd pool get mypool all
```

سپس در صورت اطمینان:

```bash
ceph osd pool delete mypool mypool --yes-i-really-really-mean-it
```

ساختار دستور:

```text
ceph osd pool delete <pool_name> <pool_name> --yes-i-really-really-mean-it
```

مثال:

```bash
ceph osd pool delete old-test-pool old-test-pool --yes-i-really-really-mean-it
```

> **هشدار جدی:** حذف Pool یک عملیات مخرب است و می‌تواند باعث از بین رفتن دائمی داده‌ها شود. این دستور را در Production بدون بررسی کامل وابستگی‌ها اجرا نکنید.

---

# 20. بررسی Pool بعد از تغییر

بعد از ایجاد یا تغییر Pool:

```bash
ceph osd pool ls detail
```

سپس:

```bash
ceph osd pool get mypool all
```

و:

```bash
ceph osd pool autoscale-status
```

و در نهایت:

```bash
ceph -s
```

را بررسی کنید.

---

# 21. ساخت یک Pool کامل برای RBD

برای یک Cluster آزمایشی سه‌نودی، یک نمونه ساده:

```bash
ceph osd pool create mypool 32
```

سپس:

```bash
ceph osd pool set mypool size 3
```

```bash
ceph osd pool set mypool min_size 2
```

```bash
ceph osd pool application enable mypool rbd
```

فعال‌کردن Autoscaler:

```bash
ceph osd pool set mypool pg_autoscale_mode on
```

در صورت نیاز تعیین نسبت مورد انتظار:

```bash
ceph osd pool set mypool target_size_ratio 0.2
```

سپس بررسی:

```bash
ceph osd pool get mypool all
```

و:

```bash
ceph osd pool autoscale-status
```

---

# 22. معادل مفهومی Ansible و Ceph CLI

اگر در Ansible چنین تنظیمی داشته باشید:

```yaml
ceph.automation.ceph_pool:
  name: mypool
  state: present
  pool_type: replicated
  size: 3
  min_size: 2
  pg_autoscale_mode: on
  application: rbd
```

معادل مفهومی آن در CLI:

```bash
ceph osd pool create mypool 32
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2
ceph osd pool set mypool pg_autoscale_mode on
ceph osd pool application enable mypool rbd
```

سپس:

```bash
ceph osd pool get mypool all
```

برای Verification.

> توجه کنید که این دو روش کاملاً یکسان نیستند. Ansible یک ابزار Automation و Idempotency است، در حالی که `ceph` CLI مستقیماً دستورات مدیریتی را روی Cluster اجرا می‌کند.

---

# 23. Workflow پیشنهادی

برای مدیریت یک Pool، این ترتیب را دنبال کنید:

```text
1. بررسی Cluster
       ↓
2. بررسی OSDها
       ↓
3. بررسی Poolهای موجود
       ↓
4. ایجاد Pool
       ↓
5. تنظیم Replica / min_size
       ↓
6. تنظیم CRUSH Rule در صورت نیاز
       ↓
7. فعال‌کردن Application
       ↓
8. تنظیم PG Autoscaler
       ↓
9. بررسی autoscale-status
       ↓
10. بررسی نهایی Cluster
```

دستورات اصلی:

```bash
ceph -s
ceph osd tree
ceph osd pool ls detail
ceph osd pool create mypool 32
ceph osd pool set mypool size 3
ceph osd pool set mypool min_size 2
ceph osd pool application enable mypool rbd
ceph osd pool set mypool pg_autoscale_mode on
ceph osd pool autoscale-status
ceph osd pool get mypool all
ceph -s
```

---

# 24. مرجع سریع دستورات

| عملیات                  | دستور                                                              |
| ----------------------- | ------------------------------------------------------------------ |
| مشاهده Poolها           | `ceph osd pool ls`                                                 |
| جزئیات Poolها           | `ceph osd pool ls detail`                                          |
| جزئیات یک Pool          | `ceph osd pool get mypool all`                                     |
| ایجاد Replicated Pool   | `ceph osd pool create mypool 32`                                   |
| تنظیم Replica           | `ceph osd pool set mypool size 3`                                  |
| تنظیم Minimum Replica   | `ceph osd pool set mypool min_size 2`                              |
| فعال‌کردن RBD            | `ceph osd pool application enable mypool rbd`                      |
| فعال‌کردن CephFS         | `ceph osd pool application enable mypool cephfs`                   |
| فعال‌کردن RGW            | `ceph osd pool application enable mypool rgw`                      |
| فعال‌کردن Autoscaler     | `ceph osd pool set mypool pg_autoscale_mode on`                    |
| غیرفعال‌کردن Autoscaler  | `ceph osd pool set mypool pg_autoscale_mode off`                   |
| مشاهده Autoscaler       | `ceph osd pool autoscale-status`                                   |
| تنظیم Target Ratio      | `ceph osd pool set mypool target_size_ratio 0.2`                   |
| تنظیم PG                | `ceph osd pool set mypool pg_num 32`                               |
| مشاهده CRUSH Ruleها     | `ceph osd crush rule ls`                                           |
| تغییر CRUSH Rule        | `ceph osd pool set mypool crush_rule replicated_rule`              |
| تغییر نام Pool          | `ceph osd pool rename mypool newpool`                              |
| حذف Pool                | `ceph osd pool delete mypool mypool --yes-i-really-really-mean-it` |

---

# نکته نهایی

برای کار روزمره با Poolها، این چهار دستور بیشترین کاربرد را دارند:

```bash
ceph osd pool ls detail
```

```bash
ceph osd pool get <pool> all
```

```bash
ceph osd pool autoscale-status
```

```bash
ceph -s
```

اگر قرار است تغییری روی Pool انجام دهید، ابتدا وضعیت فعلی را مشاهده کنید، سپس تغییر را اعمال کنید و در پایان دوباره وضعیت Pool و Cluster را بررسی کنید.

> **قاعده عملی:** قبل از اجرای هر دستور `set` یا `delete`، اول وضعیت فعلی Pool را با `ceph osd pool get <pool> all` مشاهده کنید. در Production هیچ تنظیمی را صرفاً بر اساس مثال آموزشی تغییر ندهید.
