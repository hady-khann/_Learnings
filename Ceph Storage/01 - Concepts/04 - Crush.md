# CRUSH چیست؟

**CRUSH** یکی از مهم‌ترین بخش‌های معماری Ceph است.

وظیفه‌ی اصلی آن تعیین این است که یک PG باید روی کدام OSDها قرار بگیرد.

---

# مشکلی که CRUSH حل می‌کند

در Storageهای سنتی ممکن است یک سیستم مرکزی داشته باشیم که مشخص کند:

```text
Object X → OSD.7
Object Y → OSD.12
Object Z → OSD.3
```

اگر تعداد Objectها به میلیاردها برسد، چنین جدول مرکزی بسیار بزرگ می‌شود و می‌تواند تبدیل به:

* Bottleneck
* نقطه‌ی وابستگی
* مصرف‌کننده‌ی زیاد RAM
* مشکل مقیاس‌پذیری

شود.

---

# راه‌حل CRUSH

CRUSH به‌جای اینکه مکان هر Object را در یک جدول بزرگ ذخیره کند، مکان داده را **محاسبه** می‌کند.

به‌صورت ساده:

```text
Object / PG
     ↓
CRUSH Map
     ↓
CRUSH Algorithm
     ↓
OSD Set
```

در نتیجه کلاینت می‌تواند با داشتن اطلاعات لازم، مسیر رسیدن به OSDها را محاسبه کند.

---

# مزایای CRUSH

## مقیاس‌پذیری

لازم نیست برای میلیاردها Object یک جدول مرکزی از Location هر Object نگه‌داری شود.

## توزیع داده

CRUSH می‌تواند داده را بین OSDها توزیع کند.

## Failure Domain

CRUSH می‌تواند Replicaها را در Failure Domainهای مختلف قرار دهد.

مثلاً:

```text
Replica 1 → Rack A
Replica 2 → Rack B
Replica 3 → Rack C
```

بنابراین خرابی یک Rack لزوماً باعث از بین رفتن تمام Replicaها نمی‌شود.

---

# CRUSH Map چیست؟

**CRUSH Map** توپولوژی منطقی/فیزیکی کلاستر را برای CRUSH توصیف می‌کند.

یک ساختار ساده می‌تواند به شکل زیر باشد:

```text
Root
 ├── Datacenter
 │    ├── Rack
 │    │    ├── Host
 │    │    │    ├── OSD
 │    │    │    └── OSD
 │    │    └── Host
 │    └── Rack
 └── Datacenter
```

CRUSH Map همچنین اطلاعاتی مانند Weight مربوط به OSDها را در خود دارد.

Weight معمولاً ظرفیت نسبی OSD را منعکس می‌کند.

---

# Failure Domain

فرض کنیم:

```text
replica = 3
```

اگر CRUSH هیچ محدودیتی برای Failure Domain نداشته باشد، ممکن است هر سه Replica در یک Host یا Rack قرار بگیرند.

مثلاً:

```text
Rack-A
 ├── OSD.1
 ├── OSD.2
 └── OSD.3
```

در این شرایط خرابی Rack-A می‌تواند هر سه Replica را هم‌زمان تحت تأثیر قرار دهد.

با CRUSH Rule مناسب می‌توان الزام کرد که Replicaها در Failure Domainهای متفاوت قرار بگیرند:

```text
Replica 1 → Rack-A / Host-1 / OSD.1
Replica 2 → Rack-B / Host-4 / OSD.7
Replica 3 → Rack-C / Host-8 / OSD.15
```

---

# مشاهده‌ی CRUSH Map

برای دریافت CRUSH Map:

```bash
ceph osd getcrushmap
```

---

# CRUSH Hashing

CRUSH از محاسبات deterministic برای انتخاب OSDها استفاده می‌کند.

به زبان ساده:

```text
PG + CRUSH Map
       ↓
   Calculation
       ↓
OSD Selection
```

**Deterministic** یعنی با ورودی و وضعیت یکسان، نتیجه‌ی محاسبه نیز یکسان خواهد بود.

این ویژگی باعث می‌شود اجزای مختلف Ceph بتوانند بدون نگه‌داری یک جدول مرکزی Location، به نتیجه‌ی یکسان برسند.

---

# وقتی OSD اضافه یا حذف می‌شود چه اتفاقی می‌افتد؟

وقتی توپولوژی کلاستر تغییر کند، CRUSH Map نیز تغییر می‌کند.

در نتیجه mapping برخی PGها تغییر می‌کند.

CRUSH طوری طراحی شده که تغییرات باعث **حداقل جابه‌جایی منطقی داده** نسبت به یک remapping کامل شوند.

مثلاً:

```text
OSD جدید اضافه شد
       ↓
CRUSH Map تغییر کرد
       ↓
برخی PGها Mapping جدید می‌گیرند
       ↓
داده مورد نیاز منتقل می‌شود
       ↓
کلاستر دوباره به حالت متعادل می‌رسد
```

---

# رابطه‌ی سه مفهوم

```text
CRUSH Map
    ↓
توصیف توپولوژی و منابع کلاستر

CRUSH Algorithm
    ↓
اعمال قوانین توزیع و Failure Domain

CRUSH Calculation / Hashing
    ↓
محاسبه‌ی سریع Mapping
```

به زبان ساده:

> **CRUSH Map می‌گوید کلاستر چه شکلی است.**

> **CRUSH Rule مشخص می‌کند داده چگونه باید توزیع شود.**

> **CRUSH Algorithm محاسبه می‌کند هر PG روی کدام OSDها قرار بگیرد.**

---

# نکته‌ی مهم

CRUSH مستقیماً برای هر Object یک تصمیم مستقل و جداگانه نمی‌گیرد.

مسیر مفهومی در Ceph بیشتر به این شکل است:

```text
Object
  ↓
Pool
  ↓
PG
  ↓
CRUSH
  ↓
Acting Set
  ↓
OSD
  ↓
Physical Disk
```

این مدل یکی از پایه‌های اصلی مقیاس‌پذیری Ceph است.
