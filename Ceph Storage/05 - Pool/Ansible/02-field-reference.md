# مرجع فیلدهای `ceph.automation.ceph_pool`

> **مرجع اصلی:** برای مشاهده مستندات دقیق و متناسب با نسخه نصب‌شده روی همان Node، همیشه ابتدا اجرا کنید:
>
> ```bash
> ansible-doc ceph.automation.ceph_pool
> ```

---

## `name`

**الزامی**

نام Pool را مشخص می‌کند.

```yaml
name: mypool
```

مثال:

```yaml
name: rbd
```

این فیلد مشخص می‌کند Pool با چه نامی ایجاد یا مدیریت شود.

---

## `state`

مشخص می‌کند چه عملیاتی روی Pool انجام شود.

### `present`

اگر Pool وجود نداشته باشد، آن را ایجاد یا مدیریت می‌کند.

```yaml
state: present
```

این حالت معمولاً حالت پیش‌فرض است.

### `absent`

Pool را حذف می‌کند.

```yaml
state: absent
```

> **هشدار بسیار مهم:** حذف Pool می‌تواند باعث از بین رفتن داده‌های آن شود. قبل از استفاده در Production حتماً وضعیت داده‌ها و وابستگی‌های Pool را بررسی کنید.

### `list`

برای مشاهده Poolهای موجود استفاده می‌شود و برای ایجاد یا حذف Pool نیست.

```yaml
state: list
```

---

## `pool_type`

نوع Pool را مشخص می‌کند.

دو حالت اصلی:

```text
replicated
erasure
```

---

## `replicated`

در این نوع Pool، داده به‌صورت Replicaهای کامل روی OSDهای مختلف ذخیره می‌شود.

مثال:

```yaml
pool_type: replicated
size: 3
```

یعنی هر Object دارای 3 Replica است.

این نوع Pool برای بسیاری از کاربردهای معمول مناسب است، از جمله:

* RBD
* دیسک ماشین‌های مجازی
* بسیاری از Workloadهای عمومی CephFS

برای یک محیط آموزشی یا آزمایشی، `replicated` معمولاً انتخاب ساده‌تر و قابل‌فهم‌تری است.

---

## `erasure`

در Erasure Coding، داده به چند Data Chunk تقسیم شده و Chunkهای Parity نیز تولید می‌شوند.

مزیت اصلی:

* مصرف فضای کمتر نسبت به Replicaهای کامل

معایب:

* پیچیدگی بیشتر
* پردازش بیشتر
* مناسب نبودن برای تمام Workloadها

برای مثال، می‌تواند برای مواردی مانند موارد زیر مناسب‌تر باشد:

* آرشیو
* Object Storage
* داده‌های حجیم
* Workloadهایی که کاهش مصرف Storage اهمیت بیشتری از Latency پایین دارد

> برای RBD و محیط آموزشی کوچک، معمولاً `replicated` انتخاب عملی‌تری است.

---

## `size`

مخصوص Poolهای `replicated`.

تعداد Replicaهای هر Object را مشخص می‌کند.

```yaml
size: 3
```

یعنی هر Object روی 3 OSD مختلف نگهداری می‌شود.

این عدد شامل Replica اصلی نیز می‌شود.

اگر مشخص نشود، Ceph از تنظیمات پیش‌فرض کلاستر استفاده می‌کند، مانند:

```text
osd_pool_default_size
```

در یک کلاستر آزمایشی 3 نودی، مقدار زیر متداول است:

```yaml
size: 3
```

---

## `min_size`

مخصوص Poolهای `replicated`.

حداقل تعداد Replicaهای سالم موردنیاز برای ادامه سرویس‌دهی I/O را مشخص می‌کند.

مثال:

```yaml
size: 3
min_size: 2
```

یعنی Pool در شرایطی که حداقل 2 Replica سالم باقی مانده باشد، می‌تواند همچنان I/O را سرویس دهد.

یک الگوی متداول:

```text
min_size = size - 1
```

اما مقدار مناسب به طراحی کلاستر و سیاست تحمل خرابی بستگی دارد.

> مقدار `min_size` را صرفاً بر اساس یک فرمول ثابت انتخاب نکنید. رفتار موردنظر هنگام Failure را مشخص کنید و سپس مقدار را تعیین کنید.

---

## `erasure_profile`

مخصوص Poolهای `erasure`.

نام Erasure Code Profile مورد استفاده را مشخص می‌کند.

مثال:

```yaml
erasure_profile: default
```

Profile تعیین می‌کند چند Chunk مربوط به Data و چند Chunk مربوط به Parity باشد.

برای مثال:

```text
k = 4
m = 2
```

یعنی:

```text
4 Data Chunks
2 Parity Chunks
```

این فیلد فقط زمانی اهمیت دارد که:

```yaml
pool_type: erasure
```

باشد.

---

## `rule_name`

نام CRUSH Rule مورد استفاده توسط Pool.

مثال:

```yaml
rule_name: replicated_rule
```

CRUSH Rule تعیین می‌کند داده چگونه روی Storage hierarchy توزیع شود.

می‌توان از Ruleهای متفاوت برای سناریوهایی مانند موارد زیر استفاده کرد:

* تفکیک HDD و SSD
* توزیع Replicaها بین Hostها
* Rack Awareness
* کنترل محل قرارگیری Replicaها

اگر مقدار مشخص نشود، بسته به نوع Pool و تنظیمات کلاستر، Rule مناسب/پیش‌فرض مورد استفاده قرار می‌گیرد.

> مقدار دقیق Default را برای نسخه نصب‌شده از `ansible-doc` بررسی کنید.

---

## مرجع سریع

| Field             | کاربرد                                   |
| ----------------- | ---------------------------------------- |
| `name`            | نام Pool                                 |
| `state`           | ایجاد، حذف یا عملیات مربوط به وضعیت Pool |
| `pool_type`       | نوع Pool                                 |
| `size`            | تعداد Replicaها در Poolهای replicated    |
| `min_size`        | حداقل Replica سالم برای I/O              |
| `erasure_profile` | Erasure Code Profile                     |
| `rule_name`       | CRUSH Rule                               |

برای ادامه فیلدها، به فایل‌های مربوط به PG، Autoscaler و Application مراجعه کنید.
