<div dir="rtl" lang="fa">

# تنظیم نرم‌افزار: cache، فایل‌های باز، و PG

جلسهٔ آنلاین نهم (Iran Linux House، ۲۷ نوامبر ۲۰۲۲) با اسلایدهای **Software tuning** شروع می‌شود. اول روی `ceph-node1` پوشهٔ `~/ceph-ansible/group_vars` را باز کردند (`ceph_stable_release: pacific` در `all.yml.sample`)؛ آن بخش مرور نصب است و در `02 - Install` مانده. اینجا فقط پارامترهای اسلاید ۵۲ تا ۵۴ را نگه دارید.

OpenStack این جلسه روی کلاستر دیگری است (`14 - OpenStack`). دیاگرام دو شبکهٔ Client/Cluster همان اسلاید جلسهٔ هشتم است؛ تکرار نمی‌شود — `11 - Hardware`.

## MDS و RBD cache

اسلاید ۵۲:

```text
mds cache size = 250000
rbd cache size = 67108864
```

`67108864` بایت یعنی **۶۴ MiB** کش کلاینت RBD. عدد MDS تعداد inode در cache است، نه بایت.

این‌ها مقدار پیشنهادی دوره برای لاب آموزشی‌اند، نه حد سخت هر نسخه. در کلاسترهای جدیدتر خیلی از این‌ها را با `ceph config set` می‌گذارند، نه فقط `ceph.conf`.

## max open files

اگر این پارامتر در کانفیگ Ceph باشد و کلاستر بالا بیاید، سقف file descriptor در سطح OS هم بالا می‌رود تا OSD به `too many open files` نخورد.

اسلاید:

```text
max open files = 131072
```

در چت کلاس معادل‌های سیستم‌عامل هم آمد (اسلاید آن‌ها را اجرا نکرد):

```bash
ulimit -n 10000
```

در چت یک‌بار `unlimit` تایپ شد؛ دستور درست `ulimit` است.

پایدار در `/etc/sysctl.conf`:

```text
fs.file-max = 2097152
```

موقت:

```bash
echo 26234859 > /proc/sys/fs/file-max
```

`26234859` رقم چت است؛ با `2097152` اسلاید یکی نیست. برای ماندگار بودن همان sysctl را بگذارید.

## osd pool default min size

اسلاید ۵۳: در حالت degrade، حداقل چند replica باید زنده باشد تا نوشتن از کلاینت acknowledge شود. باید از `osd pool default size` کوچک‌تر باشد. اگر به این کف نرسید، Ceph نوشتن را تأیید نمی‌کند.

```text
default value is 0
osd pool default min size = 2
```

اسلاید می‌گوید پیش‌فرض ۰ است و برای production که یکپارچگی داده مهم است مقدار **۲** را بگذارید. در نسخه‌های جدیدتر پیش‌فرض معمولاً صفر نیست؛ رقم اسلاید را با `ceph config get osd osd_pool_default_min_size` روی کلاستر خودتان چک کنید.

## تعداد PG پیش‌فرض

اسلاید ۵۴: حدود **۱۰۰ PG به‌ازای هر OSD** را هدف بگیرید.

```text
(Total number of OSD * 100) / number of replicas
```

مثال اسلاید: ۱۰ OSD و replica ۳:

```text
(10 * 100) / 3 = 333
```

پس تعداد PG باید **زیر ۳۳۳** باشد. مقدار پیش‌فرض اسلاید:

```text
osd pool default pg num = 128
```

در همین جلسه روی کلاستر hoodadcloud ساخت Pool با `pg_num 128` برای `vms` به سقف `mon_max_pg_per_osd` خورد؛ جزئیات در `14 - OpenStack`. فرمول اسلاید تخمین دوره است؛ autoscaler در `06 - Pool` است.
</div>
