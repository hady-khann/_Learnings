# مدیریت دیسک‌ها و اضافه کردن OSD در Ceph

این راهنما نحوه بررسی دیسک‌های قابل استفاده برای OSD و اضافه کردن تمام دیسک‌های مناسب یا فقط یک دیسک مشخص را نشان می‌دهد.

> **هشدار:** اضافه کردن یک Disk به عنوان OSD معمولاً باعث می‌شود Ceph آن Disk را برای ذخیره‌سازی اختصاصی استفاده کند. قبل از اجرای دستورات، مطمئن شوید Disk حاوی اطلاعات مهم یا Filesystem موردنیاز نیست.

## 1. بررسی تمام Diskهای قابل استفاده

برای مشاهده وضعیت Diskهای موجود در کلاستر:

```bash
ceph orch device ls
```

برای نمایش جزئیات بیشتر:

```bash
ceph orch device ls --wide
```

نمونه خروجی:

```text
HOST         PATH      TYPE  SIZE  DEVICE  AVAIL  REJECT REASONS
ceph-node1   /dev/sdb  hdd   100G  0       Yes
ceph-node1   /dev/sdc  hdd   100G  0       Yes
ceph-node1   /dev/sdd  hdd   100G  0       Yes
```

ستون `AVAIL` نشان می‌دهد که آیا Ceph این Disk را برای OSD قابل استفاده می‌داند یا خیر.

## 2. بررسی وضعیت OSDهای فعلی

قبل از اضافه کردن OSD، وضعیت فعلی کلاستر را بررسی کنید:

```bash
ceph -s
```

و:

```bash
ceph osd tree
```

## 3. اضافه کردن تمام Diskهای قابل استفاده

برای اینکه Ceph تمام Diskهای مناسب یک Host را به عنوان OSD استفاده کند:

```bash
ceph orch apply osd --all-available-devices
```

این دستور به Ceph می‌گوید تمام Deviceهایی که برای OSD قابل استفاده هستند را به‌صورت خودکار به OSD تبدیل کند.

بعد از اجرا بررسی کنید:

```bash
ceph orch device ls
```

و:

```bash
ceph osd tree
```

همچنین وضعیت کلی کلاستر را بررسی کنید:

```bash
ceph -s
```

> **نکته مهم:** `--all-available-devices` یک سیاست declarative است. یعنی Cephadm دستگاه‌های جدیدی را که بعداً به‌عنوان Device قابل استفاده شناسایی شوند نیز می‌تواند به OSD تبدیل کند. برای محیط Production، اگر می‌خواهید کنترل دقیق‌تری داشته باشید، اضافه کردن Disk به‌صورت صریح گزینه بهتری است.

## 4. اضافه کردن فقط یک Disk

اگر فقط می‌خواهید یک Disk مشخص را به OSD تبدیل کنید، از Device مشخص استفاده کنید.

مثلاً:

```bash
ceph orch daemon add osd ceph-node1:/dev/sdb
```

در این مثال فقط:

```text
ceph-node1:/dev/sdb
```

به عنوان OSD اضافه می‌شود.

## 5. بررسی OSD جدید

بعد از اضافه کردن Disk:

```bash
ceph orch ps --daemon-type osd
```

یا:

```bash
ceph osd tree
```

برای بررسی Device:

```bash
ceph orch device ls
```

و در نهایت:

```bash
ceph -s
```

## 6. اضافه کردن یک Disk روی Node دیگر

برای مثال، اگر Disk موردنظر روی `ceph-node2` باشد:

```bash
ceph orch daemon add osd ceph-node2:/dev/sdb
```

برای Node سوم:

```bash
ceph orch daemon add osd ceph-node3:/dev/sdb
```

## خلاصه دستورات

### مشاهده Diskها

```bash
ceph orch device ls
```

### اضافه کردن تمام Diskهای قابل استفاده

```bash
ceph orch apply osd --all-available-devices
```

### اضافه کردن فقط یک Disk

```bash
ceph orch daemon add osd ceph-node1:/dev/sdb
```

### بررسی OSDها

```bash
ceph osd tree
```

### بررسی وضعیت کلاستر

```bash
ceph -s
```

### بررسی Daemonهای OSD

```bash
ceph orch ps --daemon-type osd
```
