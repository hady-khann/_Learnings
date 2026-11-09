# CephFS از صفر + راه‌اندازی CephFS Mirroring بین دو کلاستر (cephadm)

این مستند دو بخش دارد:

- **بخش اول:** CephFS چیست و چطور از صفر روی یک کلاستر cephadm راه‌اندازی و استفاده می‌شود (مفاهیم، ساخت filesystem، کاربر، mount، subvolume، quota).
- **بخش دوم:** mirroring یک‌طرفه از `site-a` (مبدأ) به `site-b` (مقصد). مبتنی بر snapshot است و فقط snapshot ها منتقل می‌شوند. این یک DR است، نه backup.

اگر CephFS از قبل روی هر دو کلاستر کار می‌کند، مستقیم برو سراغ بخش دوم.

---

# بخش اول: CephFS از صفر

## ۱. مفاهیم

### سه نوع ذخیره‌سازی در Ceph

| نوع | سرویس | شبیه به | کاربرد |
|---|---|---|---|
| Block | RBD | یک دیسک خام | دیسک VM، دیتابیس |
| File | CephFS | یک NFS/شیر فایل با دایرکتوری | داده مشترک بین چند سرور |
| Object | RGW | S3 | بکاپ، آرشیو، اپلیکیشن |

هر سه روی یک لایه مشترک به نام **RADOS** ساخته می‌شوند: OSD ها داده را نگه می‌دارند و Ceph داده را در **pool** ها تقسیم می‌کند.

### CephFS چیست؟

یک فایل‌سیستم POSIX توزیع‌شده است. چند کلاینت می‌توانند همزمان آن را mount کنند و مثل یک دیسک محلی با دایرکتوری و فایل و permission کار کنند.

### اجزای اصلی

| جزء | توضیح |
|---|---|
| **MDS** (Metadata Server) | نام فایل‌ها، ساختار دایرکتوری‌ها، permission و اندازه فایل‌ها را مدیریت می‌کند. محتوای فایل از MDS رد نمی‌شود؛ کلاینت داده را مستقیم از OSD ها می‌خواند و می‌نویسد. |
| **metadata pool** | متادیتا (در عمل omap روی OSD) اینجا ذخیره می‌شود. کوچک است، ولی به سرعت حساس است؛ ایده‌آل SSD/NVMe. |
| **data pool** | محتوای واقعی فایل‌ها. |
| **filesystem** | ترکیب یک metadata pool + یک یا چند data pool + MDS ها. اسم پیش‌فرض در این مستند `cephfs` است. |
| **client** | kernel client (دستور `mount -t ceph`، سریع‌تر و پیشنهادی) یا ceph-fuse (userspace، انعطاف بیشتر، کندتر). |

### Active و Standby در MDS

- هر MDS یک **rank** می‌گیرد. تعداد rank های فعال با `max_mds` تعیین می‌شود (پیش‌فرض ۱).
- MDS های اضافه **standby** هستند و اگر active بمیرد جایش را می‌گیرند. پس حداقل ۲ MDS داشته باش.
- `standby-replay`: standby ای که لحظه‌به‌لحظه journal active را دنبال می‌کند و failover سریع‌تری دارد.

### cephx و caps

هر کلاینت یک کاربر cephx می‌گیرد (مثل `client.app1`) که مشخص می‌کند به کدام filesystem و کدام مسیر چه دسترسی‌ای دارد. دستور `ceph fs authorize` caps لازم برای mon و mds و osd را یک‌جا می‌سازد. حروف دسترسی: `r` خواندن، `w` نوشتن، `p` تغییر layout و quota، `s` ساخت snapshot.

### Subvolume

لایه مدیریتی بالای دایرکتوری: هر subvolume یک دایرکتوری ایزوله با quota و کاربر مجزاست و داخل یک subvolumegroup قرار می‌گیرد. ابزارهایی مثل Ceph-CSI (برای Kubernetes) و OpenStack Manila از همین استفاده می‌کنند. اگر فقط چند دایرکتوری ساده لازم داری، subvolume ضروری نیست.

### Snapshot

داخل هر دایرکتوری CephFS یک دایرکتوری مجازی `.snap` هست. ساخت `mkdir .snap/نام` یک snapshot از همان دایرکتوری (و زیردایرکتوری‌هایش) می‌سازد، آنی و بدون کپی داده. mirroring هم دقیقاً همین snapshot ها را منتقل می‌کند.

### Quota

محدودیت اندازه و تعداد فایل روی هر دایرکتوری، با xattr تنظیم می‌شود (مرحله ۸).

---

## ۲. پیش‌نیازها

روی نود ادمین:

```bash
ceph -s
ceph osd tree
ceph orch device ls
```

- `ceph -s` باید `HEALTH_OK` بدهد و OSD ها `up` و `in` باشند.
- برای production حداقل ۳ نود با OSD، و حداقل ۲ نود برای MDS (یکی active، یکی standby).
- pool های پیش‌فرض replica با `size=3` هستند؛ با کمتر از ۳ نود/OSD سالم کلاستر degraded می‌ماند.
- روی نود ادمین `ceph-common` نصب باشد (مثل مرحله ۰ بخش دوم):

```bash
cephadm install ceph-common
```

## ۳. ساخت filesystem

ساده‌ترین راه، که pool ها و MDS ها را خودش می‌سازد:

```bash
ceph fs volume create cephfs --placement="2"
```

- `cephfs` نام filesystem است.
- `--placement="2"` یعنی ۲ daemon از نوع MDS (یکی active، یکی standby). برای انتخاب نودهای مشخص: `--placement="2 node1 node2"`.
- دو pool خودکار ساخته می‌شود: `cephfs.cephfs.meta` و `cephfs.cephfs.data`.

### روش دستی (اگر کنترل بیشتر روی pool ها می‌خواهی)

```bash
ceph osd pool create cephfs_metadata
ceph osd pool create cephfs_data
ceph fs new cephfs cephfs_metadata cephfs_data
ceph orch apply mds cephfs --placement="2"
```

## ۴. بررسی

```bash
ceph fs ls
ceph fs status cephfs
ceph orch ps --daemon-type mds
ceph mds stat
```

خروجی سالم `ceph fs status`: یک MDS با وضعیت `active` و یکی `standby`. اگر `up:creating` یا `failed` دیدی چند ثانیه صبر کن و دوباره بزن.

تنظیمات مفید:

```bash
ceph fs get cephfs | grep -E "max_mds|standby"
ceph fs set cephfs max_mds 1
ceph fs set cephfs allow_standby_replay true
ceph osd pool ls detail | grep cephfs
```

- `max_mds`: تعداد MDS فعال. بیشتر کردنش فقط وقتی لازم است که بار متادیتا (تعداد زیاد فایل و دایرکتوری) زیاد باشد.
- `allow_standby_replay`: failover سریع‌تر، به قیمت مصرف بیشتر منابع.

## ۵. ساخت کاربر cephx برای کلاینت

هر برنامه/سرور یا Subvolume بهتر است کاربر CephX مستقل داشته باشد؛ مثلاً `client.app1` برای `app/app1` و `client.db1` برای `db/db1`. این کار دسترسی‌ها را از هم جدا می‌کند و اصل **حداقل دسترسی** را رعایت می‌کند.

### دسترسی به کل filesystem

فقط وقتی لازم است کل filesystem در اختیار کلاینت باشد:

```bash
ceph fs authorize cephfs client.adminfs / rw
```

### دسترسی به یک مسیر مشخص

```bash
ceph fs authorize cephfs client.app1 /data rw
```

برای ساخت snapshot نیز `s` را اضافه کن (`rws`) و برای تغییر layout یا quota، در صورت نیاز `p` را هم بده.

### دسترسی اختصاصی به یک Subvolume

ابتدا مسیر واقعی Subvolume را از خود Ceph بگیر؛ مسیر شامل UUID است و نباید آن را حدس بزنی:

```bash
ceph fs subvolume getpath cephfs app1 --group_name app
```

سپس همان مسیر را به `ceph fs authorize` بده و برای آن Subvolume یک کاربر جدا بساز:

```bash
ceph fs authorize cephfs client.app1 \
  /volumes/app/app1/<UUID> rw
```

برای Subvolume دیگر، نام کاربر و مسیر خودش را استفاده کن؛ برای نمونه:

```bash
ceph fs subvolume getpath cephfs db1 --group_name db
ceph fs authorize cephfs client.db1 \
  /volumes/db/db1/<UUID> rw
```

> `<UUID>` را با خروجی واقعی `getpath` جایگزین کن. مسیرهای `ceph fs authorize` از ریشه filesystem هستند. از یک کاربر مشترک برای چند Subvolume استفاده نکن، مگر اینکه عمداً دسترسی مشترک می‌خواهی.

بررسی دسترسی:

```bash
ceph auth get client.app1
ceph auth get client.db1
```

در خروجی باید `caps mds` شامل `fsname=cephfs` و `path=` همان Subvolume باشد. اگر برای کاربر موجود دوباره `authorize` اجرا می‌کنی، ابتدا caps فعلی را بررسی کن؛ ممکن است دسترسی‌های قبلی آن کاربر تغییر کند.

## ۶. آماده‌سازی فایل‌های احراز هویت کلاینت

برای **Kernel mount**، فایل secret که فقط خود کلید را دارد کافی است. روی نود ادمین:

```bash
ceph config generate-minimal-conf > /tmp/ceph.conf
ceph auth get-key client.app1 > /tmp/app1.secret
chmod 600 /tmp/app1.secret
```

برای **Ceph-FUSE**، به‌جای secret، کل Keyring را دریافت کن:

```bash
ceph auth get client.app1 -o /tmp/ceph.client.app1.keyring
chmod 600 /tmp/ceph.client.app1.keyring
```

فایل‌های لازم را با `scp` به کلاینت منتقل کن. نمونه برای FUSE:

```bash
scp /tmp/ceph.conf /tmp/ceph.client.app1.keyring root@<CLIENT-IP>:/root/
```

روی کلاینت:

```bash
apt install -y ceph-common ceph-fuse
mkdir -p /etc/ceph
mv /root/ceph.conf /etc/ceph/ceph.conf
mv /root/ceph.client.app1.keyring /etc/ceph/ceph.client.app1.keyring
chmod 600 /etc/ceph/ceph.client.app1.keyring
```

- کلاینت باید به Monitorها (پورت‌های 3300 و 6789)، OSDها (معمولاً 6800 تا 7568) و MDSها دسترسی شبکه داشته باشد.
- فایل `app1.secret` فقط کلید خام است؛ فایل `ceph.client.app1.keyring` شامل کلید و نام کاربر است.
- برای هر کاربر جدا، Keyring جداگانه منتقل کن؛ کلید `client.db1` را به کلاینت `app1` نده.

## ۷. Mount

روی کلاینت:

```bash
mkdir -p /mnt/cephfs
mount -t ceph app1@.cephfs=/ /mnt/cephfs -o secretfile=/etc/ceph/app1.secret
```

ساختار `کاربر@.نام_fs=/مسیر` است (بعد از `@` نقطه می‌آید و fsid خالی می‌ماند؛ mon ها از `ceph.conf` خوانده می‌شوند). اگر نسخه kernel یا mount.ceph قدیمی است و این سینتکس را نمی‌شناسد، سینتکس قدیمی:

```bash
mount -t ceph <IP-mon>:6789:/ /mnt/cephfs -o name=app1,secretfile=/etc/ceph/app1.secret,fs=cephfs
```

بررسی:

```bash
df -h /mnt/cephfs
mount | grep ceph
```

### دایرکتوری `/data` را اگر هنوز نیست بساز

اگر کاربر محدود به `/data` است و این دایرکتوری وجود ندارد، با یک mount ادمین (روی نود ادمین) بساز:

```bash
mkdir -p /mnt/admin-cephfs
mount -t ceph :/ /mnt/admin-cephfs -o name=admin,fs=cephfs
mkdir -p /mnt/admin-cephfs/data
umount /mnt/admin-cephfs
```

سپس کلاینت را روی مسیر محدود mount کن:

```bash
mount -t ceph app1@.cephfs=/data /mnt/cephfs -o secretfile=/etc/ceph/app1.secret
```

### تست

```bash
echo "hello" > /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
```

### Mount دائمی (fstab)

```bash
echo 'app1@.cephfs=/data  /mnt/cephfs  ceph  secretfile=/etc/ceph/app1.secret,_netdev,noatime  0 0' >> /etc/fstab
mount -a
```

`_netdev` مهم است تا mount بعد از بالا آمدن شبکه انجام شود.

### جایگزین: Ceph-FUSE

اگر Kernel client قابل استفاده نیست، از Ceph-FUSE استفاده کن. ابتدا Keyring کاربر را طبق مرحله ۶ روی کلاینت قرار بده:

```bash
mkdir -p /mnt/cephfs
ceph-fuse -n client.app1 \
  --client_fs cephfs \
  /mnt/cephfs
```

برای mount کردن فقط یک مسیر مشخص، از گزینه `-r` استفاده کن؛ مثال کامل Subvolume در مرحله ۹ آمده است. اگر Keyring در مسیر پیش‌فرض نیست، آن را با `-k /path/to/keyring` معرفی کن.

## ۸. Quota

روی یک دایرکتوری mount شده (کاربر باید حرف `p` داشته باشد یا با admin):

```bash
apt install -y attr
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data    # 100 گیگابایت
setfattr -n ceph.quota.max_files -v 1000000 /mnt/cephfs/data
getfattr -n ceph.quota.max_bytes /mnt/cephfs/data
```

برای برداشتن quota مقدار را `0` بگذار. quota در سمت کلاینت اعمال می‌شود و سخت‌گیرانه (hard) نیست؛ کلاینت‌های مخرب یا قدیمی می‌توانند از آن عبور کنند.

## ۹. Subvolume، کاربر مستقل و Mount با Ceph-FUSE

Subvolumeها داخل یک filesystem قرار می‌گیرند، اما می‌توان برای هرکدام quota و کاربر CephX مستقل تعریف کرد. مثال این بخش دو Subvolume با گروه‌های جداگانه `app` و `db` است.

### ۹.۱. ساخت Group و Subvolume

```bash
ceph fs subvolumegroup create cephfs app
ceph fs subvolume create cephfs app1 --group_name app --size 10737418240

ceph fs subvolumegroup create cephfs db
ceph fs subvolume create cephfs db1 --group_name db --size 10737418240
```

`--size` برحسب بایت است؛ مقدار بالا ۱۰ GiB است.

### ۹.۲. گرفتن مسیر واقعی Subvolume

این دستور را برای هر Subvolume اجرا کن:

```bash
ceph fs subvolume getpath cephfs app1 --group_name app
ceph fs subvolume getpath cephfs db1 --group_name db
```

نمونه خروجی:

```text
/volumes/app/app1/<UUID>
/volumes/db/db1/<UUID>
```

UUID را خود Ceph تعیین می‌کند. مسیر خروجی `getpath` را عیناً در مراحل بعد استفاده کن؛ مسیر را دستی نساز یا UUID را حدس نزن.

### ۹.۳. ساخت کاربر اختصاصی برای هر Subvolume

برای `app1`:

```bash
ceph fs authorize cephfs client.app1 \
  /volumes/app/app1/<UUID> rw
```

برای `db1`:

```bash
ceph fs authorize cephfs client.db1 \
  /volumes/db/db1/<UUID> rw
```

در هر دو دستور، `<UUID>` را با مسیر کامل واقعی که از `getpath` گرفتی جایگزین کن.

- `client.app1` فقط به مسیر Subvolume مربوط به `app1` مجوز می‌گیرد.
- `client.db1` فقط به مسیر Subvolume مربوط به `db1` مجوز می‌گیرد.
- برای هر Subvolume/برنامه از کاربر مجزا استفاده کن تا compromise شدن یک کلید، دسترسی به بقیه Subvolumeها را ندهد.
- `rw` یعنی خواندن و نوشتن. برای ساخت snapshot، در صورت نیاز `s` را هم اضافه کن (`rws`).

بررسی Caps:

```bash
ceph auth get client.app1
ceph auth get client.db1
```

در خروجی هر کاربر، `caps mds` باید `fsname=cephfs` و `path=` مسیر همان Subvolume را نشان دهد.

### ۹.۴. استخراج Keyring مجزا

روی نود ادمین، برای هر کاربر فایل Keyring جدا بساز:

```bash
ceph auth get client.app1 -o /root/ceph.client.app1.keyring
ceph auth get client.db1 -o /root/ceph.client.db1.keyring

chmod 600 /root/ceph.client.app1.keyring
chmod 600 /root/ceph.client.db1.keyring
```

Keyring مربوط به هر کاربر را امن به سروری که قرار است همان Subvolume را mount کند منتقل کن. برای مثال، اگر روی همان نود تست می‌کنی، فایل‌ها را در `/root/` نگه دار.

### ۹.۵. Mount کردن Subvolume با Ceph-FUSE

ابتدا مسیر mount را بساز. مثال برای `db1`:

```bash
mkdir -p /mnt/cephfs/db/db1
```

سپس با کاربر و Keyring اختصاصی خودش mount کن. مقدار `-r` باید دقیقاً خروجی `getpath` باشد:

```bash
ceph-fuse \
  -n client.db1 \
  -k /root/ceph.client.db1.keyring \
  -m <MON-IP>:6789 \
  --client_fs cephfs \
  -r /volumes/db/db1/<UUID> \
  /mnt/cephfs/db/db1
```

برای مثال واقعی این محیط، IP مانیتور `192.168.57.25` است؛ در محیط دیگر IP مانیتور خودت را قرار بده.

برای `app1` نیز همین مراحل را با اطلاعات خودش اجرا کن:

```bash
mkdir -p /mnt/cephfs/app/app1

ceph-fuse \
  -n client.app1 \
  -k /root/ceph.client.app1.keyring \
  -m <MON-IP>:6789 \
  --client_fs cephfs \
  -r /volumes/app/app1/<UUID> \
  /mnt/cephfs/app/app1
```

### ۹.۶. بررسی Mount و تست نوشتن

```bash
mount | grep ceph
 df -hT /mnt/cephfs/db/db1
```

تست نوشتن با همان کلاینت:

```bash
echo "cephfs db1 test" > /mnt/cephfs/db/db1/test.txt
cat /mnt/cephfs/db/db1/test.txt
```

برای اطمینان از جداسازی، کاربر `client.db1` نباید بتواند به Subvolume دیگر (مثلاً مسیر `app1`) دسترسی داشته باشد.

### ۹.۷. Unmount

قبل از unmount مطمئن شو فرایندی در مسیر mount مشغول کار نیست:

```bash
fusermount3 -u /mnt/cephfs/db/db1
```

در صورت نیاز برای Subvolume دیگر نیز مسیر خودش را unmount کن.

### نکات مهم

- مسیر mount محلی مثل `/mnt/cephfs/db/db1` با مسیر داخل CephFS مثل `/volumes/db/db1/<UUID>` فرق دارد.
- `-n` نام کاربر CephX، `-k` فایل Keyring همان کاربر، `-m` آدرس Monitor و `-r` ریشه‌ای است که کلاینت اجازه دیدن آن را دارد.
- اگر پیام `no keyring found` دیدی، مسیر/محتوای Keyring یا نام کاربر را بررسی کن.
- اگر پیام `rejected us (non-allowable root ...)` دیدی، مسیر `-r` با MDS cap کاربر مطابقت ندارد؛ خروجی `ceph auth get <user>` و `ceph fs subvolume getpath` را بررسی کن.
- اگر خطای `Operation not permitted` گرفتی، اول مطمئن شو برای همان Subvolume کاربر مجزا ساخته‌ای و مسیر کامل `getpath` را به `ceph fs authorize` داده‌ای.

حذف Subvolume (با احتیاط؛ داده‌های آن حذف می‌شود):

```bash
ceph fs subvolume rm cephfs db1 --group_name db
```

## ۱۰. Snapshot

```bash
ceph fs set cephfs allow_new_snaps true     # معمولاً به‌صورت پیش‌فرض فعال است
mkdir /mnt/cephfs/data/.snap/snap-test
ls /mnt/cephfs/data/.snap/
rmdir /mnt/cephfs/data/.snap/snap-test      # حذف snapshot
```

برای زمان‌بندی خودکار به مرحله ۱۰ بخش دوم (snap_schedule) نگاه کن.

## ۱۱. عیب‌یابی سریع (راه‌اندازی اولیه)

| علامت | کار |
|---|---|
| `ceph fs status` هیچ MDS فعالی ندارد | `ceph orch ps --daemon-type mds` و `cephadm logs --name <نام-daemon-mds>` |
| وضعیت `HEALTH_WARN` با `insufficient standby MDS` | تعداد MDS را بیشتر کن: `ceph orch apply mds cephfs --placement="3"` |
| `mount error: no mds server is up or the cluster is laggy` | اتصال به mon ها و MDS ها، فایروال، وضعیت MDS |
| `mount error 13 = Permission denied` | کلید اشتباه یا caps کاربر؛ `ceph auth get <client>` و `ceph fs subvolume getpath` را بررسی کن. برای هر Subvolume کاربر مستقل بساز. |
| `no keyring found` در Ceph-FUSE | فایل Keyring پیدا نشده یا نام/مسیر آن اشتباه است؛ گزینه `-k` و `-n` را بررسی کن. |
| `rejected us (non-allowable root ...)` | مسیر `-r` با مسیر مجاز در MDS cap یکی نیست؛ مسیر واقعی `getpath` را در `ceph fs authorize` و `-r` عیناً استفاده کن. |
| `mount error 110 = Connection timed out` | شبکه یا فایروال بین کلاینت و mon/OSD/MDS |
| mount کار می‌کند ولی نوشتن خطا می‌دهد | حرف `w` در caps؟ quota؟ پر بودن pool (`ceph df`)؟ |
| ساخت snapshot خطای مجوز می‌دهد | حرف `s` در caps کاربر، و `allow_new_snaps` |

## ۱۲. حذف کامل filesystem (فقط روی لب!)

> این عملیات تمام داده filesystem را برای همیشه پاک می‌کند.

```bash
ceph fs fail cephfs
ceph config set mon mon_allow_pool_delete true
ceph fs volume rm cephfs --yes-i-really-mean-it
ceph config set mon mon_allow_pool_delete false
```

---

# بخش دوم: CephFS Mirroring

جهت انتقال در این مثال یک‌طرفه است: از `site-a` (مبدأ) به `site-b` (مقصد). Mirroring بر اساس Snapshot کار می‌کند؛ بنابراین تغییرات فقط پس از ساخته‌شدن و انتقال Snapshot در مقصد دیده می‌شوند. این قابلیت برای DR مفید است، اما جایگزین Backup مستقل نیست.

در این سناریو، به‌جای Mirror کردن کل filesystem، دو Subvolume مشخص را Mirror می‌کنیم:

| مورد | Group | Subvolume | مسیر روی filesystem مبدأ |
|---|---|---|---|
| برنامه | `app` | `app1` | `/volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd` |
| دیتابیس | `db` | `db1` | `/volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea` |

> این UUIDها همان خروجی‌هایی هستند که در کلاستر فعلی گرفته‌ای. اگر Subvolume را حذف و دوباره ایجاد کنی، UUID جدید خواهد داشت؛ در آن صورت مسیر را دوباره با `ceph fs subvolume getpath` بگیر و در تمام مراحل Mirroring جایگزین کن.

## مقادیری که باید با مقادیر خودت جایگزین کنی

| نماد | معنی |
|---|---|
| `site-a` | نام کلاستر مبدأ |
| `site-b` | نام کلاستر مقصد |
| `<IP-site-a>` | IP نود ادمین کلاستر مبدأ |
| `<IP-site-b>` | IP نود ادمین کلاستر مقصد |
| `cephfs` | نام filesystem در هر دو کلاستر |
| مسیرهای `app1` و `db1` | مسیرهای واقعی Subvolumeهای بالا |

**مهم:** دستورهای مراحل ۲ تا ۴ و ۸ تا ۱۰ روی مبدأ اجرا می‌شوند؛ ساخت کاربر و token در مراحل ۵ و ۶ روی مقصد انجام می‌شود. قبل از اجرای هر دستور، مطمئن شو CLI به کلاستر درست متصل است (`ceph fsid`).

## مرحله ۰: آماده‌سازی (روی نود ادمین هر دو کلاستر)

روی هر دو کلاستر:

```bash
cephadm install ceph-common
apt install -y jq
ceph -s
ceph fsid
ceph fs ls
```

- نام filesystem در هر دو کلاستر باید `cephfs` باشد.
- `ceph -s` را بررسی کن؛ MDS باید فعال باشد و کلاستر باید برای انتقال داده آماده باشد.
- از مبدأ باید ارتباط شبکه‌ای با Monitorهای مقصد (پورت‌های 3300 و 6789) برقرار باشد.
- دسترسی SSH از مبدأ به مقصد نیز برای روش bootstrap مورد استفاده در این مستند لازم است.

## مرحله ۱: کنترل مسیر Subvolumeها (روی site-a)

قبل از شروع، مسیرها را از خود Ceph بگیر:

```bash
ceph fs subvolume getpath cephfs app1 --group_name app
ceph fs subvolume getpath cephfs db1 --group_name db
```

خروجی مورد انتظار در کلاستر فعلی:

```text
/volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd
/volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

این مسیرها را در مراحل بعد عیناً استفاده کن. مسیر Mount محلی مثل `/mnt/cephfs/app/app1` با مسیر داخل CephFS یکی نیست؛ در فرمان‌های `snapshot mirror add` باید مسیر داخل filesystem را بدهی.

## مرحله ۲: فعال‌سازی ماژول mirroring (روی هر دو کلاستر)

```bash
ceph mgr module enable mirroring
```

## مرحله ۳: اجرای daemon (روی site-a)

```bash
ceph orch apply cephfs-mirror
ceph orch ps --daemon-type cephfs-mirror
```

صبر کن daemon با وضعیت `running` نمایش داده شود.

## مرحله ۴: فعال‌سازی mirroring روی filesystem (روی site-a)

```bash
ceph fs snapshot mirror enable cephfs
```

## مرحله ۵: ساخت کاربر cephx برای اتصال به مقصد (روی site-b)

برای اتصال Mirror به کلاستر مقصد یک کاربر اختصاصی بساز:

```bash
ceph fs authorize cephfs client.mirror_remote / rwps
```

در این مثال کاربر Mirror روی کل filesystem مقصد مجوز دارد؛ این کاربر را برای Mount عادی برنامه یا دیتابیس استفاده نکن. کاربران `client.app1` و `client.db1` همچنان کاربران مستقل و محدود به Subvolumeهای خودشان هستند.

بررسی:

```bash
ceph auth get client.mirror_remote
```

## مرحله ۶: ساخت token اتصال (روی site-b)

```bash
ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json
chmod 600 /root/cephfs-token.json
```

برای مشاهده token:

```bash
jq . /root/cephfs-token.json
```

فایل token حاوی اطلاعات حساس اتصال است؛ آن را در پیام‌رسان یا محل عمومی قرار نده.

## مرحله ۷: انتقال token از مقصد به مبدأ

این دستور را روی `site-b` اجرا کن:

```bash
scp /root/cephfs-token.json root@<IP-site-a>:/root/cephfs-token.json
```

توجه: token روی مقصد ساخته می‌شود و به مبدأ منتقل می‌شود؛ چون daemon مبدأ باید بتواند به filesystem مقصد وصل شود.

## مرحله ۸: Import کردن token (روی site-a)

```bash
chmod 600 /root/cephfs-token.json
TOKEN=$(jq -r .token /root/cephfs-token.json)
ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
unset TOKEN
```

بررسی peer:

```bash
ceph fs snapshot mirror peer_list cephfs
```

باید peer مقصد با نام `site-b` نمایش داده شود.

## مرحله ۹: اضافه‌کردن Subvolumeها به Mirroring (روی site-a)

هر Subvolume را با مسیر واقعی خودش جداگانه اضافه کن:

```bash
ceph fs snapshot mirror add cephfs \
  /volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd

ceph fs snapshot mirror add cephfs \
  /volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

بررسی وضعیت:

```bash
ceph fs snapshot mirror status cephfs
```

اگر لازم است مسیرها را دوباره از کلاستر بگیری و جایگزین کنی:

```bash
APP_PATH=$(ceph fs subvolume getpath cephfs app1 --group_name app)
DB_PATH=$(ceph fs subvolume getpath cephfs db1 --group_name db)
printf 'APP_PATH=%s\nDB_PATH=%s\n' "$APP_PATH" "$DB_PATH"
```

سپس مقدار خروجی را در دو دستور `snapshot mirror add` قرار بده. این روش از اشتباه گرفتن مسیر Mount محلی با مسیر داخلی CephFS جلوگیری می‌کند.

## مرحله ۱۰: ساخت Snapshot در مسیرهای مبدأ

Mirroring بدون Snapshot چیزی برای انتقال ندارد. برای تست، ابتدا روی مبدأ یک mount مدیریتی از ریشه filesystem انجام بده. روی نود ادمین `site-a`:

```bash
mkdir -p /mnt/cephfs-source
mount -t ceph :/ /mnt/cephfs-source -o name=admin,fs=cephfs
```

دو مسیر محلی متناظر با Subvolumeها:

```bash
APP_MOUNT=/mnt/cephfs-source/volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd
DB_MOUNT=/mnt/cephfs-source/volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

برای تست، در هر Subvolume یک فایل بساز:

```bash
echo "test app1" > "$APP_MOUNT/mirror-test.txt"
echo "test db1" > "$DB_MOUNT/mirror-test.txt"
```

سپس در هر مسیر Snapshot بساز:

```bash
mkdir "$APP_MOUNT/.snap/snap1"
mkdir "$DB_MOUNT/.snap/snap1"
```

بررسی Snapshotها:

```bash
ls "$APP_MOUNT/.snap/"
ls "$DB_MOUNT/.snap/"
```

برای استفاده دائمی می‌توانی Snapshot Schedule جداگانه برای هر مسیر تنظیم کنی:

```bash
ceph mgr module enable snap_schedule
ceph fs snap-schedule add /volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd 1h
ceph fs snap-schedule add /volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea 1h
```

نمونه retention برای نگهداری ۲۴ Snapshot ساعتی:

```bash
ceph fs snap-schedule retention add /volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd h 24
ceph fs snap-schedule retention add /volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea h 24
```

وضعیت Scheduleها:

```bash
ceph fs snap-schedule status /volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd
ceph fs snap-schedule status /volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

## مرحله ۱۱: بررسی انتقال روی مقصد

روی `site-a` وضعیت daemon و peer را بررسی کن:

```bash
ceph fs snapshot mirror daemon status
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror status cephfs
```

بعد از چند ثانیه تا چند دقیقه، روی نود ادمین `site-b` filesystem را mount کن:

```bash
mkdir -p /mnt/cephfs-dr
mount -t ceph :/ /mnt/cephfs-dr -o name=admin,fs=cephfs
```

مسیرهای متناظر روی مقصد:

```bash
APP_DR=/mnt/cephfs-dr/volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd
DB_DR=/mnt/cephfs-dr/volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

Snapshotها و فایل‌های تست را بررسی کن:

```bash
ls "$APP_DR/.snap/"
ls "$DB_DR/.snap/"
cat "$APP_DR/.snap/snap1/mirror-test.txt"
cat "$DB_DR/.snap/snap1/mirror-test.txt"
```

در Mirroring مبتنی بر Snapshot، داده Snapshot در مقصد بررسی می‌شود؛ انتظار نداشته باش هر تغییر جدید قبل از ساخته‌شدن و Sync شدن Snapshot در مقصد ظاهر شود.

## مرحله ۱۲: پاک‌سازی فایل token و Unmount

پس از اطمینان از برقراری peer، فایل token را از هر دو کلاستر پاک کن:

```bash
shred -u /root/cephfs-token.json
```

Mountهای تست را نیز در صورت عدم نیاز باز کن:

```bash
umount /mnt/cephfs-source
umount /mnt/cephfs-dr
```

---

## عیب‌یابی سریع (Mirroring)

| علامت | بررسی / راهکار |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps --daemon-type cephfs-mirror` و `cephadm logs --name <نام-daemon>` روی مبدأ |
| import خطا می‌دهد | نام filesystem، اعتبار token، دسترسی شبکه به Monitorهای مقصد و ساعت دو کلاستر |
| مسیر به Mirroring اضافه نمی‌شود | مسیر را با `ceph fs subvolume getpath` بگیر؛ مسیر داخلی CephFS را بده، نه `/mnt/...` را |
| Snapshot منتقل نمی‌شود | در مسیر درست `.snap` ساخته شده؟ مسیر با `snapshot mirror add` اضافه شده؟ `ceph fs snapshot mirror status cephfs` چه می‌گوید؟ |
| authentication error | `ceph auth get client.mirror_remote` روی مقصد و caps آن را بررسی کن |
| فایل در مقصد دیده نمی‌شود | ابتدا داخل `.snap/<snapshot-name>/` بررسی کن؛ Mirroring Snapshotمحور است و لزوماً فایل جاری را همان لحظه نشان نمی‌دهد |

## حذف مسیرها و peer

برای توقف Mirror کردن هر Subvolume، روی مبدأ مسیر مربوط را جداگانه حذف کن:

```bash
ceph fs snapshot mirror remove cephfs \
  /volumes/app/app1/30014e96-1757-480d-ae5b-997477fcfabd

ceph fs snapshot mirror remove cephfs \
  /volumes/db/db1/aaa2ebad-1407-4922-98f1-668559eefdea
```

برای حذف peer (فقط پس از حذف مسیرهای Mirror شده و اطمینان از عدم نیاز):

```bash
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror peer_remove cephfs <peer-uuid>
```

برای غیرفعال‌کردن Mirroring روی filesystem:

```bash
ceph fs snapshot mirror disable cephfs
```

## نکات مهم

- روی مقصد در مسیرهایی که Mirror می‌شوند به‌صورت مستقل داده ننویس؛ مبدأ مرجع داده است.
- این Mirroring یک‌طرفه است؛ failover و failback را باید با برنامه‌ریزی و به‌صورت دستی انجام بدهی.
- فقط Snapshotها منتقل می‌شوند؛ فاصله Snapshotها و مدت زمان Sync، RPO را تعیین می‌کند.
- کاربرهای برنامه و دیتابیس (`client.app1` و `client.db1`) برای Mount مستقل باقی می‌مانند. کاربر `client.mirror_remote` فقط برای ارتباط Mirroring بین دو کلاستر است.
