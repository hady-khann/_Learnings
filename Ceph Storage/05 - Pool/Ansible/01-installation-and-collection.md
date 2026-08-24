# نصب و شناسایی Collection مربوط به Ceph Pool

## نکته مهم

ماژول Pool در این پروژه از Collection زیر استفاده می‌شود:

```text
ceph.automation
```

بنابراین نام کامل ماژول:

```text
ceph.automation.ceph_pool
```

است.

> **هشدار:** قبلاً تصور می‌شد ماژول `ceph_pool` متعلق به `community.general` است. این فرض صحیح نیست. همیشه Collection نصب‌شده روی همان سیستم را بررسی کنید و صرفاً بر اساس مستندات قدیمی یا مثال‌های اینترنتی تصمیم نگیرید.

---

## 1. بررسی نسخه Ansible

ابتدا مطمئن شوید نسخه `ansible-core` مناسب و مورد استفاده مشخص است.

در این محیط نسخه مورد استفاده:

```text
ansible-core 2.21.3
```

برای بررسی:

```bash
ansible --version
```

در صورت وجود مشکل در packageهای Ansible می‌توان وضعیت packageها را بررسی کرد:

```bash
apt-mark showhold
```

اگر لازم بود:

```bash
sudo apt-mark unhold ansible ansible-core
sudo apt remove --purge -y ansible
sudo apt install -y ansible-core
sudo apt-mark hold ansible-core
```

> قبل از اجرای دستورات حذف یا تغییر packageها، وضعیت فعلی سیستم را بررسی کنید. این دستورات صرفاً یک الگوی عملی برای محیط موردنظر هستند.

---

## 2. نصب Collection

Collection را به‌صورت **System-wide** نصب کنید تا برای تمام کاربران و Administratorهای سیستم در دسترس باشد.

مسیر پیشنهادی:

```text
/usr/share/ansible/collections
```

نصب:

```bash
sudo ansible-galaxy collection install ceph.automation \
  -p /usr/share/ansible/collections \
  --timeout 120
```

---

## 3. بررسی Collection نصب‌شده

بعد از نصب:

```bash
ansible-galaxy collection list | grep -i ceph
```

باید Collection مربوط به Ceph را مشاهده کنید.

---

## 4. بررسی مستندات واقعی ماژول

مهم‌ترین مرحله این است که مستندات همان نسخه نصب‌شده را مستقیماً از Ansible بخوانید:

```bash
ansible-doc ceph.automation.ceph_pool
```

این دستور مرجع اصلی برای موارد زیر است:

* نام فیلدها
* نوع فیلدها
* Required بودن
* مقدار پیش‌فرض
* مقادیر مجاز
* رفتار ماژول در نسخه نصب‌شده

> **قاعده اصلی این پروژه:** اگر بین این فایل‌های راهنما و خروجی `ansible-doc` اختلافی وجود داشت، خروجی `ansible-doc` اولویت دارد.

---

## 5. بررسی مسیر Collection

برای پیدا کردن محل نصب Collection:

```bash
ansible-galaxy collection list | grep -i ceph
```

همچنین می‌توانید مسیرهای Collection را بررسی کنید:

```bash
ansible-config dump | grep COLLECTIONS_PATH
```

مسیر System-wide معمولاً شامل:

```text
/usr/share/ansible/collections
```

است.

---

## خلاصه

قبل از استفاده از ماژول:

```bash
ansible-galaxy collection list | grep -i ceph
```

سپس:

```bash
ansible-doc ceph.automation.ceph_pool
```

و در Playbook:

```yaml
ceph.automation.ceph_pool:
```

استفاده کنید.
