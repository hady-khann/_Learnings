# انواع Storage در Ceph

Ceph سه مدل اصلی ارائه می‌دهد:

```text
Object Storage → RGW
Block Storage  → RBD
File Storage   → CephFS
```

---

# Object Storage — RGW

**RADOS Gateway (RGW)** برای Object Storage استفاده می‌شود.

داده به‌صورت Object ذخیره می‌شود و معمولاً شامل:

```text
Data + Metadata + Unique Identifier
```

است.

ساختار آن شبیه سرویس‌هایی مانند Amazon S3 است.

مناسب برای:

* Backup
* Archive
* فایل‌های وب
* Object Storage
* داده‌های غیرساختاریافته

---

# Block Storage — RBD

**RADOS Block Device (RBD)** یک Block Device مجازی در اختیار کلاینت قرار می‌دهد.

سیستم‌عامل می‌تواند آن را مانند یک دیسک معمولی ببیند:

```text
Ceph RBD
   ↓
Virtual Disk
   ↓
Filesystem
   ↓
VM / Server
```

برای مواردی مانند:

* ماشین‌های مجازی
* دیتابیس
* OpenStack Cinder
* دیسک‌های VM

مناسب است.

---

# File Storage — CephFS

**CephFS** یک فایل‌سیستم توزیع‌شده است.

کلاینت می‌تواند آن را Mount کند و مانند یک فایل‌سیستم معمولی از آن استفاده کند:

```text
CephFS
 ├── directory
 ├── file
 └── subdirectory
```

CephFS برای مدیریت Metadata به **MDS** نیاز دارد.

---

# RADOS

**RADOS — Reliable Autonomic Distributed Object Store**

RADOS هسته‌ی اصلی Storage در Ceph است.

مدل‌های مختلف Ceph در نهایت روی RADOS قرار می‌گیرند:

```text
RGW
 │
 ├── RADOS
 │
RBD
 │
 └── RADOS

CephFS
 │
 └── RADOS
```

RADOS مسئول قابلیت‌هایی مانند:

* توزیع داده
* Replication
* Recovery
* تحمل خرابی
* مدیریت Objectها

است.

---

# librados

**librados** یک API/Library برای دسترسی مستقیم برنامه‌ها به RADOS است.

به کمک آن یک Application می‌تواند بدون استفاده از RBD یا RGW مستقیماً با RADOS کار کند.

```text
Application
     ↓
librados
     ↓
RADOS
     ↓
OSD
```

این قابلیت بیشتر برای توسعه‌دهندگانی کاربرد دارد که می‌خواهند مستقیماً با Storage Engine داخلی Ceph کار کنند.
