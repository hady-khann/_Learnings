<div dir="rtl" lang="fa">

# دیسک، نسبت SSD به OSD، و شاسی

ادامهٔ همان اسلایدهای جلسهٔ هشتم: نوع دیسک journal/WAL، نسبت فلش به HDD، و مثال شاسی SuperMicro / HPE.

## نسبت SSD یا NVMe به OSD داده

دیسک **HDD** در نوشتن‌های کوچک (journal / WAL) کند است. Ceph نوشتن را اول روی فلش سریع ثبت می‌کند (**WAL** = write-ahead log، **DB** = RocksDB متادیتای BlueStore؛ در FileStore قدیمی به آن journal می‌گفتند)، بعد به‌صورت پشت‌سرهم روی HDD می‌ریزد. فلش را بین چند OSD HDD شریک می‌کنید تا هزینه پایین بماند.

اسلاید Disk:

```text
SATA/SAS SSD  →  نسبت 1:4   (یک SSD برای چهار دیسک داده)
PCIe / NVMe   →  نسبت 1:12 تا 1:18  (بسته به کارایی فلش)
```

یعنی یک دستگاه فلش را به‌عنوان WAL/DB (یا journal قدیمی) بین چند OSD HDD شریک می‌کنید. NVMe سریع‌تر است پس OSDهای بیشتری روی همان دستگاه جا می‌شود.

اگر نسبت را خیلی بالا ببرید، صف WAL پر می‌شود و latency نوشتن بالا می‌رود. ۱:۴ برای SSD و ۱:۱۲ برای NVMe حد تقریبی اسلاید است: NVMe IOPS بیشتری دارد پس OSD بیشتری روی همان دستگاه جا می‌شود. از آن بیشتر، چند HDD همزمان WAL را قفل می‌کنند.

در چت کلاس سرعت‌های تقریبی آمد:

| رسانه | رقم کلاس |
|---|---|
| SAS ۱۵K RPM | حدود ۱۵۰۰۰ IOPS |
| SAS ۱۰K RPM | حدود ۱۰۰۰۰ IOPS |
| SSD | حدود ۴۰۰۰۰ IOPS |

این‌ها ترتیب بزرگی برای بحث هستند، نه بنچمارک لاب.

## DAS در برابر SAN

سؤال کلاس: آیا OSD را روی LUN سن می‌گذارند؟ بحث **DAS** در برابر SAN بود.

خود **Ceph** replication می‌کند (`size=3`). اگر OSD را روی LUN پشت RAID/SAN بگذارید، دو لایه کپی دارید، کنترلر خرابی دیسک را از Ceph پنهان می‌کند، و CRUSH نمی‌داند Failure Domain واقعی کجاست. مدل رایج: دیسک محلی (**DAS** / JBOD / HBA passthrough) تا هر OSD یک دیسک فیزیکی باشد. جزئیات CRUSH در پوشهٔ `01 - Concepts` است.

## مثال شاسی SuperMicro

اسلاید ۴۵ یک رک ۴۲U برای Object Storage نشان می‌دهد:

```text
320 TB     1 PB      2 PB     روی SuperRack
10GbE Switches:  48× 10GbE ports / 24× 10GbE ports
Monitor Node:    4× 3.5" HDD bays
OSD Nodes:       12× / 36× / 72×  3.5" HDD bays
```

ایده: MON را روی شاسی کوچک جدا بگذارید؛ OSD را روی شاسی پر از HDD.

## HPE DL380 و NVMe

استاد صفحهٔ پیکربندی HPE ProLiant DL380 Gen10 را باز کرد (ترکیب NVMe + SAS/SATA). چت کلاس پرسید NVMe روی DL20 Gen10 و M.2. این بخش کاتالوگ است، نه نصب لاب.

برای ادامهٔ نرم‌افزار (`mds cache size`، `osd_journal_size`، jumbo frame) جلسهٔ نهم را ببینید؛ آن یادداشت جدا نوشته می‌شود.
</div>
