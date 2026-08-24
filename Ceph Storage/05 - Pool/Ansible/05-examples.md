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
