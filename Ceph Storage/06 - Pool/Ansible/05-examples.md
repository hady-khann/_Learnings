# نمونه‌های `ceph.automation.ceph_pool`

## نمونه 1 — Pool ساده Replicated برای RBD

این رایج‌ترین الگو برای یک محیط آموزشی یا آزمایشی است:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure mypool exists
      ceph.automation.ceph_pool:
        name: mypool
        state: present
        pool_type: replicated
        size: 3
        application: rbd
```

---

## نمونه 2 — Replicated Pool با PG Autoscaler

در این روش تعداد PG را به‌صورت دستی تعیین نمی‌کنیم:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure mypool exists with autoscaling
      ceph.automation.ceph_pool:
        name: mypool
        state: present
        pool_type: replicated
        size: 3
        pg_autoscale_mode: on
        target_size_ratio: 0.2
        application: rbd
```

در این مثال:

```text
Pool type        → replicated
Replica count    → 3
PG Autoscaler    → on
Expected share   → 20%
Application      → RBD
```

---

## نمونه 2.1 — Replicated Pool با `target_size_bytes`

وقتی حجم دقیق دادهٔ آینده Pool را می‌دانید (نه سهم نسبی آن از کل کلاستر)، به‌جای `target_size_ratio` می‌توانید از `target_size_bytes` استفاده کنید:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure backup-pool exists with a fixed target size
      ceph.automation.ceph_pool:
        name: backup-pool
        state: present
        pool_type: replicated
        size: 3
        pg_autoscale_mode: on
        target_size_bytes: 500G
        application: rbd
```

در این مثال:

```text
Pool type        → replicated
Replica count    → 3
PG Autoscaler    → on
Expected size    → 500G
Application      → RBD
```

> نکته: `target_size_ratio` و `target_size_bytes` نباید هم‌زمان روی یک Pool تنظیم شوند؛ فقط یکی از این دو پارامتر را در Task قرار دهید.

---

## نمونه 2.2 — Replicated Pool با `bulk`

اگر نه سهم نسبی و نه حجم دقیق Pool را از قبل نمی‌دانید، اما می‌دانید این Pool قرار است حجم قابل‌توجهی داده داشته باشد (مثل یک Pool اصلی RBD یا CephFS)، فعال کردن `bulk` ساده‌ترین گزینه است:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure mypool exists as a bulk pool
      ceph.automation.ceph_pool:
        name: mypool
        state: present
        pool_type: replicated
        size: 3
        pg_autoscale_mode: on
        bulk: true
        application: rbd
```

در این مثال:

```text
Pool type        → replicated
Replica count    → 3
PG Autoscaler    → on
Bulk             → true
Application      → RBD
```

با `bulk: true`، Autoscaler از همان ابتدا فرض می‌کند Pool قرار است داده زیادی داشته باشد و تعداد PG کامل‌تری تخصیص می‌دهد؛ بدون نیاز به تعیین `target_size_ratio` یا `target_size_bytes`.

---

## نمونه 3 — Erasure-Coded Pool

برای Workloadهایی که Erasure Coding برای آن‌ها مناسب است:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure archive-pool exists
      ceph.automation.ceph_pool:
        name: archive-pool
        state: present
        pool_type: erasure
        erasure_profile: default
        pg_autoscale_mode: on
        application: rgw
```

> قبل از استفاده از `erasure_profile: default` بررسی کنید که Profile موردنظر واقعاً در Cluster وجود دارد.

برای بررسی:

```bash
ceph osd erasure-code-profile ls
```

---

## نمونه 4 — حذف Pool

```yaml
- name: Remove old test pool
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Delete old test pool
      ceph.automation.ceph_pool:
        name: old-test-pool
        state: absent
```

> **هشدار:** حذف Pool یک عملیات مخرب است. این Task را بدون بررسی دقیق در Production اجرا نکنید.

---

## نمونه پیشنهادی برای Cluster آزمایشی سه‌نودی

برای یک Cluster کوچک با 3 Node و Workload RBD:

```yaml
- name: Manage Ceph pools
  hosts: localhost
  connection: local
  become: true

  tasks:

    - name: Ensure RBD pool exists
      ceph.automation.ceph_pool:
        name: mypool
        state: present
        pool_type: replicated
        size: 3
        min_size: 2
        pg_autoscale_mode: on
        application: rbd
```

این تنظیمات به‌صورت مفهومی یعنی:

```text
Pool name       → mypool
Pool type       → replicated
Replica count   → 3
Minimum copies  → 2
PG Autoscaler   → enabled
Application     → RBD
```

---

## قبل از اجرای Playbook

ابتدا Collection را بررسی کنید:

```bash
ansible-galaxy collection list | grep -i ceph
```

سپس مستندات نسخه نصب‌شده را بخوانید:

```bash
ansible-doc ceph.automation.ceph_pool
```

> نکته: پارامترهایی مانند `target_size_bytes` و `bulk` ممکن است بسته به نسخه‌ی `ceph.automation` Collection نصب‌شده، پشتیبانی نشوند یا نام متفاوتی داشته باشند. حتماً قبل از استفاده، خروجی `ansible-doc` را برای نسخه نصب‌شده بررسی کنید.

و وضعیت Cluster را بررسی کنید:

```bash
ceph -s
```

در نهایت وضعیت Poolها:

```bash
ceph osd pool ls detail
```

و وضعیت Autoscaler:

```bash
ceph osd pool autoscale-status
```

را بررسی کنید.

> مثال‌های این فایل الگوهای آموزشی هستند. قبل از استفاده در Production باید پارامترهای Pool، Failure Domain، CRUSH Rule، Replica Size، `min_size` و PG Autoscaler متناسب با طراحی واقعی Cluster بررسی شوند.
