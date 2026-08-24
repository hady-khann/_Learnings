# Application و Cluster

## `application`

Application مورد استفاده Pool را مشخص می‌کند.

از نظر مفهومی مشابه فعال‌کردن Application روی Pool با دستور Ceph است:

```bash
ceph osd pool application enable <pool> <application>
```

مقادیر رایج:

```text
rbd
cephfs
rgw
```

---

## `rbd`

برای Block Storage و Imageهای ماشین‌های مجازی.

مثال:

```yaml
application: rbd
```

این گزینه برای Poolهایی که قرار است توسط RBD استفاده شوند، انتخاب رایجی است.

---

## `cephfs`

برای CephFS.

```yaml
application: cephfs
```

در بسیاری از سناریوها، هنگام راه‌اندازی CephFS، Application به‌صورت خودکار تنظیم می‌شود.

بنابراین معمولاً نیازی نیست آن را بدون دلیل به‌صورت دستی مدیریت کنید.

---

## `rgw`

برای Object Storage و Ceph Object Gateway.

```yaml
application: rgw
```

در سناریوهای S3-compatible Object Storage کاربرد دارد.

---

# `cluster`

نام Ceph Cluster را مشخص می‌کند.

در نصب‌های معمول نام Cluster:

```text
ceph
```

است.

در این حالت معمولاً نیازی به تعیین آن در هر Task نیست.

مثال:

```yaml
cluster: ceph
```

در محیط‌های معمول، از جمله Cluster مبتنی بر Vagrant مورد استفاده در این پروژه، می‌توان این فیلد را معمولاً حذف کرد.

---

# انتخاب Application بر اساس Workload

| Workload                | Application |
| ----------------------- | ----------- |
| VM Disk / Block Storage | `rbd`       |
| CephFS                  | `cephfs`    |
| S3 / Object Storage     | `rgw`       |

مثال RBD:

```yaml
- name: Create RBD pool
  ceph.automation.ceph_pool:
    name: rbd-pool
    state: present
    pool_type: replicated
    size: 3
    application: rbd
```

> نوع Application باید با نحوه استفاده واقعی از Pool مطابقت داشته باشد. صرفاً اضافه‌کردن `rbd` یا `rgw` بدون اینکه Pool واقعاً توسط آن سرویس استفاده شود، طراحی مناسبی نیست.
