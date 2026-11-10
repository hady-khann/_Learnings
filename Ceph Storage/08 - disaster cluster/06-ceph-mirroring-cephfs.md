# CephFS از صفر + راه‌اندازی CephFS Mirroring بین دو کلاستر (cephadm)

این مستند دو بخش دارد:

- **بخش اول:** CephFS چیست و چطور از صفر روی یک کلاستر cephadm راه‌اندازی و استفاده می‌شود (مفاهیم، ساخت filesystem، کاربر، mount، subvolume، quota).
- **بخش دوم:** mirroring یک‌طرفه از `site-a` (مبدأ) به `site-b` (مقصد). مبتنی بر snapshot است و فقط snapshot ها منتقل می‌شوند. این یک DR است، نه backup.

اگر CephFS از قبل روی هر دو کلاستر کار می‌کند، مستقیم برو سراغ بخش دوم.

> **نکته مهم (Subvolume و Mirroring):**  
> مشکل قبلی «محدودیت Ceph» نبود؛ **مسیر اشتباه** بود. `ceph fs subvolume getpath` مسیر داخلی داده (`/volumes/<group>/<subvol>/<UUID>`) را می‌دهد، ولی **ریشهٔ subvolume** مسیر `/volumes/<group>/<subvol>` است.  
> - Snapshot subvolume روی **ریشهٔ subvolume** ساخته می‌شود، نه روی UUID.  
> - مسیری که باید با `ceph fs snapshot mirror add` اضافه شود: **`/volumes/<group>/<subvol>`**.  
> - `mkdir .snap/...` داخل UUID به‌خاطر طراحی MDS همیشه `EPERM` می‌دهد (حتی برای admin)، ولی روی ریشهٔ subvolume کار می‌کند.  
> جزئیات و اثبات در بخش «Subvolume و Mirroring (روش صحیح)».

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
| **client** | kernel client (دستور `mount -t ceph`، سریع‌تر) یا ceph-fuse (userspace، انعطاف بیشتر). در محیط‌های تست/cephadm گاهی kernel mount با secret مشکل دارد؛ در آن صورت از ceph-fuse استفاده کن. |

### Active و Standby در MDS

- هر MDS یک **rank** می‌گیرد. تعداد rank های فعال با `max_mds` تعیین می‌شود (پیش‌فرض ۱).
- MDS های اضافه **standby** هستند و اگر active بمیرد جایش را می‌گیرند. پس حداقل ۲ MDS داشته باش.
- `standby-replay`: standby ای که لحظه‌به‌لحظه journal active را دنبال می‌کند و failover سریع‌تری دارد.

### cephx و caps

هر کلاینت یک کاربر cephx می‌گیرد (مثل `client.app1`) که مشخص می‌کند به کدام filesystem و کدام مسیر چه دسترسی‌ای دارد. دستور `ceph fs authorize` caps لازم برای mon و mds و osd را یک‌جا می‌سازد. حروف دسترسی: `r` خواندن، `w` نوشتن، `p` تغییر layout و quota، `s` ساخت snapshot.

### Subvolume

لایه مدیریتی بالای دایرکتوری: هر subvolume یک دایرکتوری ایزوله با quota و کاربر مجزاست و داخل یک subvolumegroup قرار می‌گیرد. ابزارهایی مثل Ceph-CSI (برای Kubernetes) و OpenStack Manila از همین استفاده می‌کنند. اگر فقط چند دایرکتوری ساده لازم داری، subvolume ضروری نیست.

> برای **Mirroring** روی Subvolume، مسیر mirror باید **ریشهٔ subvolume** (`/volumes/<group>/<subvol>`) باشد، نه مسیر `getpath` (که UUID دارد). بخش «Subvolume و Mirroring (روش صحیح)» را ببین.

### Snapshot

داخل هر دایرکتوری CephFS یک دایرکتوری مجازی `.snap` هست. ساخت `mkdir .snap/نام` یک snapshot از همان دایرکتوری (و زیردایرکتوری‌هایش) می‌سازد، آنی و بدون کپی داده. **mirroring دقیقاً همین نوع snapshot را منتقل می‌کند** (نام تمیز، بدون پیشوند `_`).

### Quota

محدودیت اندازه و تعداد فایل روی هر دایرکتوری، با xattr تنظیم می‌شود (مرحله ۸).

---

## ۲. پیش‌نیازها

روی نود ادمین:

```bash
# وضعیت کلی کلاستر را نشان می‌دهد؛ بعداً قبل از mirror هم باید HEALTH_OK باشد
ceph -s
# درخت نودها و OSDها را نشان می‌دهد؛ برای اطمینان از up و in بودن OSDها
ceph osd tree
# دیسک‌های آزاد نودها را لیست می‌کند؛ برای اطمینان از وجود دیسک کافی برای OSD
ceph orch device ls
```

- `ceph -s` باید `HEALTH_OK` بدهد و OSD ها `up` و `in` باشند.
- برای production حداقل ۳ نود با OSD، و حداقل ۲ نود برای MDS (یکی active، یکی standby).
- pool های پیش‌فرض replica با `size=3` هستند؛ با کمتر از ۳ OSD مؤثر، کلاستر **degraded / undersized** می‌ماند و می‌تواند sync را مختل کند. در لب با ۴ OSD یا کمتر موقتاً:

```bash
# تعداد replica متادیتا pool را ۲ می‌کند (فقط لب)؛ تا با OSD کم، PG ها undersized نمانند و mirror مختل نشود
ceph osd pool set cephfs.cephfs.meta size 2
# تعداد replica data pool را ۲ می‌کند (فقط لب)؛ دلیلش مثل خط قبل
ceph osd pool set cephfs.cephfs.data size 2
# حداقل replica لازم برای I/O متادیتا pool را ۱ می‌کند (فقط لب)؛ برای ادامهٔ کار هنگام degraded بودن
ceph osd pool set cephfs.cephfs.meta min_size 1
# حداقل replica لازم برای I/O data pool را ۱ می‌کند (فقط لب)؛ دلیلش مثل خط قبل
ceph osd pool set cephfs.cephfs.data min_size 1
```

- روی نود ادمین `ceph-common` نصب باشد:

```bash
# ابزارهای ceph را روی نود نصب می‌کند؛ بعداً برای اجرای دستورات ceph و ceph-fuse لازم است
cephadm install ceph-common
```

---

## ۳. ساخت filesystem

```bash
# filesystem با نام cephfs و دو MDS با pool های خودکار می‌سازد؛ بعداً روی آن mirror می‌زنیم
ceph fs volume create cephfs --placement="2"
```

- `cephfs` نام filesystem است.
- `--placement="2"` یعنی ۲ daemon MDS. برای نود مشخص: `--placement="2 node1 node2"`.
- دو pool خودکار: `cephfs.cephfs.meta` و `cephfs.cephfs.data`.

### روش دستی

```bash
# pool متادیتا را می‌سازد (روش دستی)؛ بعداً به ceph fs new داده می‌شود
ceph osd pool create cephfs_metadata
# pool داده را می‌سازد (روش دستی)؛ بعداً به ceph fs new داده می‌شود
ceph osd pool create cephfs_data
# filesystem را از دو pool بالا می‌سازد (روش دستی)
ceph fs new cephfs cephfs_metadata cephfs_data
# دو MDS برای filesystem اجرا می‌کند (روش دستی)؛ یکی active و یکی standby
ceph orch apply mds cephfs --placement="2"
```

---

## ۴. بررسی

```bash
# لیست filesystemها را نشان می‌دهد؛ بعداً برای چک کردن اسم FS و گرفتن fscid هم استفاده می‌شود
ceph fs ls
# وضعیت MDSها و poolهای cephfs را نشان می‌دهد؛ برای اطمینان از active بودن MDS
ceph fs status cephfs
# daemonهای MDS را در cephadm لیست می‌کند؛ برای اطمینان از running بودن
ceph orch ps --daemon-type mds
# خلاصهٔ وضعیت MDS را چاپ می‌کند؛ برای چک سریع active و standby
ceph mds stat
```

```bash
# تعداد MDS فعال را ۱ می‌کند؛ بقیهٔ MDSها standby می‌مانند
ceph fs set cephfs max_mds 1
# standby-replay را روشن می‌کند؛ برای failover سریع‌تر MDS
ceph fs set cephfs allow_standby_replay true
# اجازهٔ ساخت snapshot در FS را می‌دهد؛ بدون آن mirroring و snapshot کار نمی‌کند
ceph fs set cephfs allow_new_snaps true
```

---

## ۵. ساخت کاربر cephx برای کلاینت

```bash
# کاربر cephx با دسترسی rw به کل FS می‌سازد؛ برای mount ادمینی داده
ceph fs authorize cephfs client.adminfs / rw
# یا مسیر محدود:
# کاربر cephx با دسترسی rws فقط روی مسیر /data می‌سازد؛ s اجازهٔ snapshot است
ceph fs authorize cephfs client.app1 /data rws
```

برای Subvolume:

```bash
# مسیر داده (UUID) subvolume را نشان می‌دهد؛ بعداً برای ساخت caps کاربر و mount لازم است
ceph fs subvolume getpath cephfs app1 --group_name app
# کاربر cephx را محدود به مسیر داده subvolume می‌کند؛ اپ فقط همین مسیر را می‌بیند
ceph fs authorize cephfs client.app1 /volumes/app/app1/<UUID> rws
```

---

## ۶. فایل‌های احراز هویت

```bash
# فایل ceph.conf حداقلی می‌سازد؛ بعداً کلاینت برای اتصال به mon ها لازم دارد
ceph config generate-minimal-conf > /tmp/ceph.conf
# secret کاربر app1 را در فایل می‌ریزد؛ بعداً برای mount کرنل لازم است
ceph auth get-key client.app1 > /tmp/app1.secret
# دسترسی فایل secret را فقط برای root می‌کند؛ برای امنیت
chmod 600 /tmp/app1.secret

# keyring کاربر app1 را ذخیره می‌کند؛ بعداً برای ceph-fuse لازم است
ceph auth get client.app1 -o /tmp/ceph.client.app1.keyring
# دسترسی keyring را فقط برای root می‌کند؛ برای امنیت
chmod 600 /tmp/ceph.client.app1.keyring
```

---

## ۷. Mount

### Kernel (گاهی در محیط cephadm با خطای secret fail می‌شود)

```bash
# دایرکتوری mount را می‌سازد؛ بعداً FS اینجا mount می‌شود
mkdir -p /mnt/cephfs
# FS را با kernel client mount می‌کند؛ اگر خطای secret داد از ceph-fuse استفاده کن
mount -t ceph app1@.cephfs=/ /mnt/cephfs -o secretfile=/etc/ceph/app1.secret
```

اگر دیدی:

```text
adding ceph secret key to kernel failed: Invalid argument
```

از **ceph-fuse** استفاده کن.

### Ceph-FUSE (پیشنهادی در تست / وقتی kernel مشکل دارد)

```bash
# ceph-fuse را نصب می‌کند؛ بعداً برای mount در cephadm و mount ادمین مبدأ و مقصد لازم است
apt install -y ceph-fuse
# دایرکتوری mount را می‌سازد؛ بعداً FS اینجا mount می‌شود
mkdir -p /mnt/cephfs
# FS را با ceph-fuse و کاربر app1 mount می‌کند؛ جایگزین kernel mount
ceph-fuse -n client.app1 -k /etc/ceph/ceph.client.app1.keyring \
  --client_fs cephfs /mnt/cephfs
```

با root محدود (`-r`):

```bash
# فقط یک مسیر مشخص (مثلاً subvolume) را با ceph-fuse mount می‌کند؛ برای اپ محدود به داده خودش
ceph-fuse -n client.app1 -k /path/to/keyring --client_fs cephfs \
  -r /volumes/app/app1/<UUID> /mnt/cephfs/app/app1 -o allow_other
```

---

## ۸. Quota

```bash
# ابزار setfattr را نصب می‌کند؛ بعداً برای quota لازم است
apt install -y attr
# سقف حجم دایرکتوری را ۱۰۰ گیگ می‌کند؛ برای کنترل مصرف
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data
# سقف تعداد فایل دایرکتوری را یک میلیون می‌کند؛ برای کنترل مصرف
setfattr -n ceph.quota.max_files -v 1000000 /mnt/cephfs/data
```

---

## ۹. Subvolume (مدیریت داده / CSI)

Subvolume برای ایزوله‌سازی و quota خوب است و با mirroring هم کار می‌کند، به شرط اینکه **ریشهٔ subvolume** را mirror کنی (نه مسیر UUID). ساختار آن:

```text
/volumes/<group>/<subvol>/            ← ریشهٔ subvolume (xattr: ceph.dir.subvolume=1) ← مسیر mirror و محل snapshot
/volumes/<group>/<subvol>/.meta       ← متادیتای subvolume
/volumes/<group>/<subvol>/<UUID>/     ← داده (خروجی getpath) ← mount / کاربر cephx اینجاست
/volumes/<group>/<subvol>/.snap/<name>← snapshotهای subvolume
```

```bash
# subvolumegroup می‌سازد؛ بعداً subvolumeها داخل آن قرار می‌گیرند
ceph fs subvolumegroup create cephfs app
# subvolume با سقف ۱۰ گیگ می‌سازد؛ بعداً ریشهٔ آن را mirror می‌کنیم
ceph fs subvolume create cephfs app1 --group_name app --size 10737418240
# مسیر داده subvolume را می‌دهد؛ ریشهٔ mirror یک سطح بالاتر از UUID است
ceph fs subvolume getpath cephfs app1 --group_name app     # → /volumes/app/app1/<UUID>  (مسیر داده)
# مشخصات subvolume (مسیر، quota، features) را نشان می‌دهد؛ برای چک نسخه v2 و مسیر
ceph fs subvolume info cephfs app1 --group_name app
```

ساخت snapshot (محلی / CSI / **و قابل mirror**، وقتی ریشهٔ subvolume mirror شده باشد):

```bash
# snapshot subvolume را روی ریشهٔ آن می‌سازد؛ همین snapshot بعداً توسط mirror منتقل می‌شود
ceph fs subvolume snapshot create cephfs app1 snap1 --group_name app
# snapshotهای subvolume را لیست می‌کند؛ برای اطمینان از ساخته شدن
ceph fs subvolume snapshot ls cephfs app1 --group_name app
```

> این snapshot در `/volumes/app/app1/.snap/snap1` ساخته می‌شود. اگر از داخل مسیر UUID به `.snap` نگاه کنی، همان snapshot با نام `_snap1_<ino>` (snapshot «ارثی» از والد) دیده می‌شود و `cephfs-mirror` عمداً نام‌های شروع‌شده با `_` را نادیده می‌گیرد. برای همین، mirror روی مسیر UUID هیچ‌چیز sync نمی‌کند.

---

## ۱۰. Snapshot روی دایرکتوری معمولی (روش سازگار با Mirroring)

```bash
# اجازهٔ ساخت snapshot در FS را می‌دهد؛ بدون آن mkdir .snap کار نمی‌کند
ceph fs set cephfs allow_new_snaps true
# دایرکتوری معمولی تست را می‌سازد؛ بعداً snapshot روی آن گرفته می‌شود
mkdir -p /mnt/cephfs-source/data
# FS را mount می‌کند (جای ... را با روش mount خودت پر کن)؛ برای دسترسی به .snap
mount ...   # یا ceph-fuse با admin

# با mkdir داخل .snap یک snapshot می‌سازد؛ همین روش برای mirror استفاده می‌شود
mkdir /mnt/cephfs-source/data/.snap/snap-test
# snapshotهای موجود را لیست می‌کند؛ برای اطمینان از ساخته شدن
ls /mnt/cephfs-source/data/.snap/
# snapshot تست را حذف می‌کند؛ برای تمیزکاری
rmdir /mnt/cephfs-source/data/.snap/snap-test
```

---

# بخش دوم: CephFS Mirroring

جهت: یک‌طرفه از `site-a` (مبدأ) به `site-b` (مقصد). فقط **snapshot** منتقل می‌شود → DR است، نه backup لحظه‌ای.

## Subvolume و Mirroring (روش صحیح)

### ریشهٔ مشکل

دو مشاهدهٔ قبلی (`Operation not permitted` و `snap_map={}` / `nothing to synchronize`) هر دو یک علت دارند: مسیر mirror اشتباه (مسیر UUID) انتخاب شده بود.

| مشاهده | علت واقعی | شاهد |
|---|---|---|
| `mkdir <UUID>/.snap/x` → `EPERM` | MDS اجازهٔ snapshot روی **فرزندان** ریشهٔ subvolume را نمی‌دهد؛ فقط خود ریشه مجاز است. UUID فرزند ریشه است. | کد MDS: `is a descendent of a subvolume dir` → `-EPERM` |
| snapshot API در mirror sync نمی‌شود | snapshot در `<ریشه>/.snap/` ساخته می‌شود. از دید دایرکتوری UUID این snapshot ارثی است و با `_` شروع می‌شود. `build_snap_map` نام‌های `_*` را رد می‌کند. | کد `PeerReplayer::build_snap_map` |
| `getpath` مسیر UUID می‌دهد | `getpath` مسیر **داده** است، نه مسیر snapshot. | مستندات `fs-volumes` و کد mgr/volumes (xattr `ceph.dir.subvolume` روی `base_path`) |

### جدول مسیرها

| کار | مسیر درست | مسیر اشتباه |
|---|---|---|
| `ceph fs snapshot mirror add` | `/volumes/<group>/<subvol>` | `/volumes/<group>/<subvol>/<UUID>` |
| ساخت snapshot با `mkdir` | `<mount>/volumes/<group>/<subvol>/.snap/<name>` | `.../<UUID>/.snap/<name>` |
| ساخت snapshot با API | `ceph fs subvolume snapshot create ...` (همان ریشه) | — |
| زمان‌بندی | `ceph fs snap-schedule add / 1h --fs cephfs --subvol <subvol> --group <group>` | `--path` دستی روی UUID |
| mount برای اپ / کاربر cephx | مسیر UUID (`getpath`) | — |

> برای subvolume نسخهٔ v2، ماژول snap-schedule خودش با `--subvol` و `--group` مسیر را به ریشهٔ subvolume (`<UUID>/..`) تبدیل می‌کند.  
> Snapshot روی سطوح بالاتر (مثل `/volumes` یا `/volumes/<group>`) برای mirror مناسب نیست؛ برای هر subvolume یک مسیر mirror جدا بگذار (مطابق مثال‌های Red Hat و MicroCeph: `/volumes/_nogroup/<subvol>`).

### قواعد مهم مستندات رسمی

- فقط مسیرهای مطلق؛ مسیر نرمال می‌شود؛ مسیر از ریشهٔ CephFS شروع می‌شود (نه نقطهٔ mount).
- هر مسیر mirror‌شده: نه والد و نه فرزند مسیر دیگری mirror‌شده باشد (`EINVAL`).
- Mirror فقط snapshot را منتقل می‌کند. محتوای زندهٔ مقصد بعد از هر sync موفق برابر آخرین snapshot sync‌شده می‌شود.
- فقط فایل عادی، دایرکتوری و symlink منتقل می‌شود. Hard link در مقصد فایل جدا می‌شود.
- **مقصد را read-only فرض کن.** هیچ snapshot دستی (`mkdir .snap/x` یا `ceph fs subvolume snapshot create`) و هیچ snap-schedule روی مسیرهای mirror‌شدهٔ مقصد نساز؛ باعث خطای `invalid metadata` و `failed` شدن مسیر می‌شود.
- فقط یک daemon `cephfs-mirror` توصیه می‌شود (چند daemon تست نشده است).
- روی مقصد، mirror daemon مسیر را با `mkdirs` می‌سازد. xattr `ceph.dir.subvolume` و رکورد subvolume روی مقصد **ساخته نمی‌شود** (فقط داده و `.meta` کپی می‌شود). یعنی در مقصد داده در مسیر `/volumes/<group>/<subvol>/<UUID>/` موجود است، ولی ممکن است `ceph fs subvolume ls` آن را به‌عنوان subvolume مدیریتی نشان ندهد. برای DR از mount مستقیم مسیر استفاده کن (بخش Failover) و این رفتار را در تست خودت تأیید کن.
- بهتر است `subvolumegroup` را روی مقصد از قبل بسازی: `ceph fs subvolumegroup create cephfs <group>`. خود subvolume را روی مقصد نساز.

---

## مقادیر جایگزین

| نماد | معنی |
|---|---|
| `site-a` / node2 | کلاستر مبدأ |
| `site-b` / node3 | کلاستر مقصد |
| `cephfs` | نام filesystem در هر دو طرف |

دستورهای فعال‌سازی mirror و add مسیر → روی **مبدأ**.  
ساخت `client.mirror_remote` و token → روی **مقصد**.

---

## مرحله ۰: آماده‌سازی (هر دو کلاستر)

```bash
# ابزارهای ceph را نصب می‌کند؛ روی هر دو کلاستر برای اجرای دستورات بعدی لازم است
cephadm install ceph-common
# jq و attr و ceph-fuse را نصب می‌کند؛ jq برای خواندن token و ceph-fuse برای بررسی داده لازم است
apt install -y jq attr ceph-fuse
# سلامت کلاستر را چک می‌کند؛ قبل از mirror باید HEALTH_OK باشد
ceph -s
# fsid کلاستر را نشان می‌دهد؛ برای اطمینان از اینکه دو کلاستر متفاوت‌اند
ceph fsid
# نام FS را چک می‌کند؛ باید در هر دو کلاستر یکسان باشد (cephfs)
ceph fs ls
```

- نام FS در هر دو طرف یکسان باشد (`cephfs`).
- از مبدأ به monهای مقصد (3300 / 6789) دسترسی شبکه باشد.
- هر دو کلاستر ترجیحاً `HEALTH_OK` باشند (undersized زیاد می‌تواند نوشتن snapshot روی مقصد را مختل کند).

---

## مرحله ۱: فعال‌سازی ماژول و daemon (مبدأ)

```bash
# ماژول mirroring را در mgr فعال می‌کند؛ دستورات ceph fs snapshot mirror بدون آن کار نمی‌کنند
ceph mgr module enable mirroring
# daemon cephfs-mirror را با cephadm اجرا می‌کند؛ این daemon snapshotها را به مقصد می‌فرستد
ceph orch apply cephfs-mirror
# وضعیت daemon را نشان می‌دهد؛ صبر کن تا running شود
ceph orch ps --daemon-type cephfs-mirror
```

صبر کن تا `running` شود.

روی **هر دو** کلاستر (در صورت نیاز ماژول):

```bash
# ماژول mirroring را فعال می‌کند (فقط در صورت نیاز)؛ مثل مرحلهٔ ۱
ceph mgr module enable mirroring
```

---

## مرحله ۲: فعال‌سازی mirroring روی FS (مبدأ)

```bash
# mirroring را برای FS روشن می‌کند؛ قبل از add کردن peer و مسیر لازم است
ceph fs snapshot mirror enable cephfs
```

---

## مرحله ۳: کاربر و token روی مقصد (site-b)

```bash
# روی مقصد کاربر mirror_remote با دسترسی rwps می‌سازد؛ مبدأ با این کاربر در مقصد snapshot می‌سازد
ceph fs authorize cephfs client.mirror_remote / rwps
# caps کاربر را نشان می‌دهد؛ برای اطمینان از rwps
ceph auth get client.mirror_remote

# روی مقصد token اتصال می‌سازد؛ بعداً در مبدأ import می‌شود
ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json
# دسترسی فایل token را فقط برای root می‌کند؛ چون شامل کلید است
chmod 600 /root/cephfs-token.json
# محتوای token را خوانا چاپ می‌کند؛ برای اطمینان از سالم بودن فایل
jq . /root/cephfs-token.json
```

انتقال به مبدأ:

```bash
# token را به مبدأ می‌فرستد؛ بعداً در مرحلهٔ import لازم است
scp /root/cephfs-token.json root@<IP-site-a>:/root/cephfs-token.json
```

---

## مرحله ۴: Import peer (مبدأ)

```bash
# دسترسی token را در مبدأ هم محدود می‌کند؛ برای امنیت
chmod 600 /root/cephfs-token.json
# فقط مقدار token را در متغیر می‌گذارد؛ بعداً به import داده می‌شود
TOKEN=$(jq -r .token /root/cephfs-token.json)
# token را import می‌کند و مقصد را به‌عنوان peer ثبت می‌کند؛ از این به بعد mirror می‌تواند به مقصد وصل شود
ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
# متغیر token را پاک می‌کند؛ برای امنیت
unset TOKEN

# peerها را لیست می‌کند؛ باید site-b دیده شود و UUID آن بعداً برای status لازم است
ceph fs snapshot mirror peer_list cephfs
```

باید peer با `site-b` دیده شود.

---

## مرحله ۵: افزودن مسیرها برای mirror (روی مبدأ)

### الف) Subvolume (روش استاندارد برای محیط عملیاتی شما)

```bash
# نام group و subvolume را عوض کن
# نام FS را در متغیر می‌گذارد؛ در دستورات بعدی استفاده می‌شود
FS=cephfs
# نام subvolumegroup را در متغیر می‌گذارد؛ در دستورات بعدی استفاده می‌شود
GROUP=app
# نام subvolume را در متغیر می‌گذارد؛ در دستورات بعدی استفاده می‌شود
SUBVOL=app1

# بررسی: مسیر داده (فقط برای اطلاع) و ریشهٔ subvolume
# مسیر داده (با UUID) را نشان می‌دهد؛ فقط برای اطلاع، این را mirror نکن
ceph fs subvolume getpath $FS $SUBVOL --group_name $GROUP
# خروجی: /volumes/app/app1/<UUID>   ← این را mirror نکن

# مسیر صحیح mirror = ریشهٔ subvolume = دو سطح بالاتر از UUID
# ریشهٔ subvolume را از مسیر UUID بیرون می‌کشد؛ همان مسیر درست mirror است
SV_ROOT=$(dirname "$(ceph fs subvolume getpath $FS $SUBVOL --group_name $GROUP)")
# مسیر محاسبه‌شده را چاپ می‌کند؛ برای اطمینان از درست بودن (/volumes/group/subvol)
echo "$SV_ROOT"            # /volumes/app/app1

# ریشهٔ subvolume را برای mirror ثبت می‌کند؛ از این به بعد snapshotهایش به مقصد می‌رود
ceph fs snapshot mirror add $FS "$SV_ROOT"
# مسیرهای ثبت‌شده برای mirror را لیست می‌کند؛ برای تأیید
ceph fs snapshot mirror ls $FS
```

برای subvolume در group پیش‌فرض، مسیر `/volumes/_nogroup/<subvol>` است (بدون `--group_name`).

برای چند subvolume، همین کار را برای هرکدام تکرار کن:

```bash
# حلقه برای چند subvolume؛ مسیر ریشهٔ هرکدام را محاسبه و add می‌کند (فرمت group:subvol)
for SV in "app:app1" "db:db1"; do
  G=${SV%%:*}; N=${SV##*:}
  ROOT=$(dirname "$(ceph fs subvolume getpath cephfs $N --group_name $G)")
  ceph fs snapshot mirror add cephfs "$ROOT"
done
# مسیرهای ثبت‌شده را لیست می‌کند؛ برای تأیید همهٔ subvolumeها
ceph fs snapshot mirror ls cephfs
```

> `dirname` روی خروجی `getpath` تا وقتی subvolume نسخهٔ v2 است درست کار می‌کند (پیش‌فرض در Ceph مدرن). با `ceph fs subvolume info` فیلد `path` و `features` را می‌توانی چک کنی.

### ب) دایرکتوری معمولی (برای داده‌ای که subvolume نیست)

```bash
# روی مبدأ – mount ادمین با FUSE
# دایرکتوری mount مبدأ را می‌سازد؛ بعداً FS اینجا mount می‌شود
mkdir -p /mnt/cephfs-source
# FS را با کاربر admin mount می‌کند؛ برای ساخت دایرکتوری و snapshot روی مبدأ
ceph-fuse /mnt/cephfs-source -n client.admin --client_fs cephfs -o allow_other

# دو دایرکتوری معمولی می‌سازد؛ بعداً mirror می‌شوند
mkdir -p /mnt/cephfs-source/app1-data /mnt/cephfs-source/db1-data
# فایل تست داخل app1-data می‌نویسد؛ بعداً در مقصد دنبالش می‌گردیم
echo "app data" > /mnt/cephfs-source/app1-data/file.txt
# فایل تست داخل db1-data می‌نویسد؛ بعداً در مقصد دنبالش می‌گردیم
echo "db data"  > /mnt/cephfs-source/db1-data/file.txt

# app1-data را برای mirror ثبت می‌کند
ceph fs snapshot mirror add cephfs /app1-data
# db1-data را برای mirror ثبت می‌کند
ceph fs snapshot mirror add cephfs /db1-data
# مسیرهای ثبت‌شده را لیست می‌کند؛ برای تأیید
ceph fs snapshot mirror ls cephfs
```

---

## مرحله ۶: ساخت Snapshot (روی مبدأ)

Mirror فقط snapshotهای **با نام معمولی** (بدون پیشوند `_`) را منتقل می‌کند. نام snapshot باید در مقصد یکتا باشد؛ نام ثابت و تکراری استفاده نکن (مثلاً با تاریخ بساز).

```bash
# اجازهٔ ساخت snapshot را تضمین می‌کند؛ بدون آن snapshot ساخته نمی‌شود
ceph fs set cephfs allow_new_snaps true
```

### الف) Subvolume: روش A، با API (توصیه‌شده برای دستی و اسکریپت)

```bash
# نام snapshot یکتا با تاریخ می‌سازد؛ چون نام در مقصد باید تکراری نباشد
SNAP=snap-$(date +%Y%m%d-%H%M%S)
# snapshot subvolume را می‌سازد؛ mirror آن را در چرخهٔ بعدی منتقل می‌کند
ceph fs subvolume snapshot create cephfs app1 $SNAP --group_name app
# snapshotها را لیست می‌کند؛ برای اطمینان از ساخته شدن
ceph fs subvolume snapshot ls cephfs app1 --group_name app
```

### ب) Subvolume: روش B، با mkdir روی ریشهٔ subvolume

```bash
# mount ادمین از ریشهٔ FS
# snapshot را با mkdir روی ریشهٔ subvolume می‌سازد؛ معادل API (روی UUID خطا می‌دهد)
mkdir /mnt/cephfs-source/volumes/app/app1/.snap/snap-$(date +%Y%m%d-%H%M%S)
# snapshotهای ریشه را لیست می‌کند؛ برای تأیید
ls /mnt/cephfs-source/volumes/app/app1/.snap/
```

> این همان چیزی است که API انجام می‌دهد. اگر از داخل UUID بزنی (`.../app1/<UUID>/.snap/...`) `EPERM` می‌گیری؛ طبیعی است.

### پ) Subvolume: زمان‌بندی خودکار (توصیه‌شده برای production)

```bash
# ماژول زمان‌بندی snapshot را فعال می‌کند؛ بدون آن snap-schedule کار نمی‌کند
ceph mgr module enable snap_schedule
# هر ساعت یک snapshot خودکار از subvolume می‌سازد؛ mirror آن‌ها را منتقل می‌کند
ceph fs snap-schedule add / 1h --fs cephfs --subvol app1 --group app
# فقط ۲۴ snapshot ساعتی آخر را نگه می‌دارد؛ برای جلوگیری از انباشت
ceph fs snap-schedule retention add / h 24 --fs cephfs --subvol app1 --group app
# وضعیت زمان‌بندی را نشان می‌دهد؛ برای اطمینان از فعال بودن
ceph fs snap-schedule status / --fs cephfs --subvol app1 --group app
```

- اسم snapshotها `scheduled-<timestamp>` می‌شود و mirror آن‌ها را عادی می‌بیند.
- Retention مبدأ را طوری بگذار که snapshot هنوز-sync‌نشده حذف نشود. اگر snapshot قبل از sync پاک شود، فقط snapshotهای باقی‌مانده منتقل می‌شوند (RPO بدتر می‌شود).
- روی **مقصد** هیچ snap-schedule نگذار.

### ت) دایرکتوری معمولی

```bash
# snapshot دستی روی دایرکتوری معمولی می‌سازد؛ mirror آن را منتقل می‌کند
mkdir /mnt/cephfs-source/app1-data/.snap/snap-$(date +%Y%m%d-%H%M%S)
# snapshotها را لیست می‌کند؛ برای تأیید
ls /mnt/cephfs-source/app1-data/.snap/

# هر ساعت snapshot خودکار از /app1-data می‌سازد؛ mirror آن‌ها را منتقل می‌کند
ceph fs snap-schedule add /app1-data 1h
# فقط ۲۴ snapshot آخر را نگه می‌دارد؛ برای جلوگیری از انباشت
ceph fs snap-schedule retention add /app1-data h 24
# وضعیت زمان‌بندی را نشان می‌دهد؛ برای تأیید
ceph fs snap-schedule status /app1-data
```

### ث) Snapshot هماهنگ برای چند کلاینت (اختیاری، Squid به بعد)

Snapshot در CephFS بین چند کلاینتِ همزمان consistency قوی ندارد. برای دیتابیس‌ها یا اپ‌های چندکلاینتی، قبل از snapshot، subvolume را quiesce کن:

```bash
# I/O چند subvolume را موقتاً مکث می‌کند؛ تا snapshot بین کلاینت‌ها consistent شود
ceph fs quiesce cephfs --set-id snapset1 app/app1 --timeout 30 --expiration 30 --await
# snapshot را در زمان مکث می‌سازد؛ این نقطهٔ consistent است که mirror می‌برد
ceph fs subvolume snapshot create cephfs app1 snap-$(date +%Y%m%d-%H%M%S) --group_name app
# مکث I/O را آزاد می‌کند؛ بعد از snapshot حتماً لازم است
ceph fs quiesce cephfs --set-id snapset1 --release --await
```

مقادیر timeout و expiration را با نیاز خودت تنظیم کن. این مرحله فقط برای اپ‌هایی لازم است که consistency بین کلاینت‌ها مهم است.

---

## مرحله ۷: بررسی وضعیت mirror (مبدأ)

### peer list و daemon

```bash
# وضعیت daemon و peerها را نشان می‌دهد؛ برای اطمینان از اتصال به مقصد
ceph fs snapshot mirror daemon status
# peerها را لیست می‌کند؛ UUID آن برای دستورات status لازم است
ceph fs snapshot mirror peer_list cephfs
# مسیرهای mirror را لیست می‌کند؛ برای تأیید اینکه مسیر درست ثبت شده
ceph fs snapshot mirror ls cephfs
```

### وضعیت دقیق از admin socket

مسیر asok در cephadm طولانی است و ممکن است خطای `AF_UNIX path too long` بدهد. همیشه symlink کوتاه بساز:

```bash
# سوکت admin daemon را پیدا و یک symlink کوتاه می‌سازد؛ چون مسیر اصلی خطای AF_UNIX path too long می‌دهد
for a in $(find /var/run/ceph -name '*cephfs-mirror*.asok'); do
  if ceph --admin-daemon "$a" help 2>/dev/null | grep -q 'fs mirror peer status'; then
    ln -sf "$a" /tmp/mirror.asok
    echo "ASOK=$a"
    break
  fi
done

# عدد بعد از @ همان fscid است (در مثال 2). از اینجا بگیر:
# fscid را نشان می‌دهد؛ عدد بعد از @ در دستورات admin socket است
ceph fs get cephfs | grep fscid

# وضعیت کلی
# وضعیت کلی mirror را نشان می‌دهد؛ برای اطمینان از وجود peer و تعداد مسیرها
ceph --admin-daemon /tmp/mirror.asok fs mirror status cephfs@<FSCID>

# وضعیت هر مسیر (UUID peer را از peer_list بردار)
# وضعیت sync هر مسیر را نشان می‌دهد؛ state و last_synced_snap و snaps_synced از اینجا خوانده می‌شود
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

> در نسخه‌های جدید، خروجی `peer status` داخل `metrics/<مسیر>/peer/<uuid>` تو در تو است؛ در نسخه‌های قدیمی‌تر، مسیرها مستقیم کلید اصلی‌اند. معنی فیلدها یکسان است.

### وضعیت از طریق mgr (بدون admin socket)

اگر نسخهٔ شما این دستور را دارد (در مستندات `latest` آمده؛ با `ceph fs snapshot mirror -h` چک کن)، از هر نود ادمین کار می‌کند:

```bash
# وضعیت sync همهٔ مسیرها را از mgr نشان می‌دهد؛ جایگزین admin socket (اگر نسخه پشتیبانی کند)
ceph fs snapshot mirror status cephfs
# وضعیت sync یک مسیر را نشان می‌دهد؛ برای بررسی subvolume مشخص
ceph fs snapshot mirror status cephfs /volumes/app/app1
```

حالت‌های مسیر: `idle`، `syncing`، `failed`، و (فقط در دستور mgr) `stale`. شمارنده‌های `snaps_synced` / `snaps_deleted` / `snaps_renamed` فقط برای **session فعلی daemon** هستند و بعد از restart صفر می‌شوند؛ `last_synced_snap` پایدار است.

خروجی موفق شبیه:

```json
{
  "/app1-data": {
    "state": "idle",
    "last_synced_snap": {
      "id": 30,
      "name": "snap1",
      "sync_bytes": 46
    },
    "snaps_synced": 1,
    "snaps_deleted": 0,
    "snaps_renamed": 0
  }
}
```

اگر `snaps_synced: 0` و در لاگ `local snap_map={}` / `nothing to synchronize` دیدی، این موارد را بررسی کن:

1. مسیر mirror ریشهٔ subvolume باشد (نه UUID): `ceph fs snapshot mirror ls cephfs`
2. snapshot واقعاً در `<مسیر mirror>/.snap/` وجود داشته باشد و نامش با `_` شروع نشود.
3. `ceph fs snapshot mirror daemon status` برای FS مورد نظر peer نشان دهد.

### لاگ daemon

```bash
# لاگ daemon mirror را می‌خواند؛ برای دیدن خطاهای sync
cephadm logs --name $(ceph orch ps --daemon-type cephfs-mirror -f json | jq -r '.[0].daemon_name') 2>&1 | tail -100

# افزایش debug در صورت نیاز
# سطح debug را بالا می‌برد؛ فقط برای عیب‌یابی، بعداً برگردان
ceph config set client.cephfs-mirror debug_cephfs_mirror 20
# daemon را restart می‌کند؛ تا debug جدید اعمال شود
ceph orch daemon restart <cephfs-mirror-daemon-name>
```

عبارت‌های مفید در لاگ: `build_snap_map`, `do_sync_snaps`, `synchronizing`, `nothing to synchronize`, `connected to cluster`.

---

## مرحله ۸: بررسی روی مقصد (site-b)

```bash
# دایرکتوری mount مقصد را می‌سازد؛ برای بررسی داده روی مقصد
mkdir -p /mnt/cephfs-dr
# FS مقصد را با admin mount می‌کند؛ برای دیدن snapshotهای sync‌شده
ceph-fuse /mnt/cephfs-dr -n client.admin --client_fs cephfs -o allow_other

# Subvolume (مسیر در مقصد همان مسیر مبدأ است)
# محتوای مسیر subvolume را در مقصد نشان می‌دهد؛ برای اطمینان از وجود مسیر
ls -la /mnt/cephfs-dr/volumes/app/app1/
# snapshotهای sync‌شده در مقصد را لیست می‌کند؛ نام آخرین باید با last_synced_snap یکی باشد
ls -la /mnt/cephfs-dr/volumes/app/app1/.snap/
# داده داخل یک snapshot مقصد را نشان می‌دهد؛ برای تأیید محتوا
ls -la /mnt/cephfs-dr/volumes/app/app1/.snap/<SNAP>/<UUID>/

# دایرکتوری معمولی
# snapshotهای دایرکتوری معمولی را در مقصد لیست می‌کند
ls -la /mnt/cephfs-dr/app1-data/.snap/
# محتوای فایل داخل snapshot مقصد را می‌خواند؛ برای تأیید sync
cat /mnt/cephfs-dr/app1-data/.snap/<SNAP>/file.txt
```

در مقصد برای هر snapshot sync‌شده باید یک snapshot هم‌نام وجود داشته باشد. نام snapshot آخر باید با `last_synced_snap.name` در مبدأ یکی باشد.

دادهٔ **داخل snapshot** را چک کن؛ فایل «جاری» بدون snapshot جدید لزوماً به‌روز نمی‌شود.

---

## مرحله ۹: تست سریع end-to-end

### تست Subvolume (روی مبدأ)

```bash
# group تست می‌سازد؛ برای تست end-to-end
ceph fs subvolumegroup create cephfs testgrp
# subvolume تست می‌سازد؛ برای تست end-to-end
ceph fs subvolume create cephfs testsv --group_name testgrp --size 1073741824

# مسیر داده را در متغیر می‌گذارد؛ بعداً برای نوشتن فایل و محاسبهٔ ریشه لازم است
UUID_PATH=$(ceph fs subvolume getpath cephfs testsv --group_name testgrp)
# ریشهٔ subvolume را محاسبه می‌کند؛ همان مسیر mirror
SV_ROOT=$(dirname "$UUID_PATH")
# هر دو مسیر را چاپ می‌کند؛ برای اطمینان از محاسبهٔ درست
echo "$UUID_PATH  →  $SV_ROOT"

# نوشتن داده داخل مسیر UUID
# فایل تست داخل مسیر داده می‌نویسد؛ بعداً در مقصد دنبالش می‌گردیم
echo "SVTEST $(date)" > /mnt/cephfs-source${UUID_PATH}/data.txt

# ساخت snapshot و اضافه کردن مسیر
# snapshot تست می‌سازد؛ همین snapshot باید به مقصد برسد
ceph fs subvolume snapshot create cephfs testsv SVTEST1 --group_name testgrp
# ریشهٔ subvolume تست را برای mirror ثبت می‌کند
ceph fs snapshot mirror add cephfs "$SV_ROOT"

# چند ثانیه صبر می‌کند؛ تا mirror فرصت sync داشته باشد
sleep 20
# وضعیت sync را چک می‌کند؛ باید idle و snaps_synced ≥ 1 باشد
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

روی **مقصد**:

```bash
# نام پوشهٔ UUID داخل snapshot مقصد را می‌گیرد؛ چون در مقصد UUID همان مبدأ است
UUID_DIR=$(ls /mnt/cephfs-dr/volumes/testgrp/testsv/.snap/SVTEST1/)
# فایل تست را از snapshot مقصد می‌خواند؛ دیدن متن یعنی mirror سالم است
cat /mnt/cephfs-dr/volumes/testgrp/testsv/.snap/SVTEST1/$UUID_DIR/data.txt
```

### تست دایرکتوری معمولی (الگوی قبلی)

روی **مبدأ**:

```bash
# دایرکتوری تست معمولی می‌سازد
mkdir -p /mnt/cephfs-source/newtest
# فایل تست می‌نویسد؛ بعداً در مقصد دنبالش می‌گردیم
echo "NEWTESTSNAP $(date)" > /mnt/cephfs-source/newtest/data.txt
# snapshot تست می‌سازد؛ همین باید به مقصد برسد
mkdir /mnt/cephfs-source/newtest/.snap/NEWTESTSNAP
# دایرکتوری تست را برای mirror ثبت می‌کند
ceph fs snapshot mirror add cephfs /newtest
# چند ثانیه صبر می‌کند؛ تا mirror فرصت sync داشته باشد
sleep 15
# وضعیت sync را چک می‌کند؛ باید snaps_synced ≥ 1 باشد
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

روی **مقصد**:

```bash
# فایل تست را از snapshot مقصد می‌خواند؛ دیدن متن یعنی mirror سالم است
cat /mnt/cephfs-dr/newtest/.snap/NEWTESTSNAP/data.txt
```

اگر متن را دیدی، زنجیرهٔ mirror سالم است.

---

## مرحله ۱۰: پاک‌سازی token و unmount

```bash
# فایل token را امن پاک می‌کند؛ چون دیگر بعد از import لازم نیست و کلید دارد
shred -u /root/cephfs-token.json
# fusermount3 -u /mnt/cephfs-source
# fusermount3 -u /mnt/cephfs-dr
```

---

## Checkpoint (اختیاری، برای اطمینان از رسیدن snapshot مهم)

در مستندات `latest` آمده؛ با `ceph fs snapshot mirror -h` وجود آن را در نسخهٔ خودت چک کن.

```bash
# یک snapshot مهم را checkpoint می‌کند؛ بعداً وضعیت رسیدنش به مقصد را ببینیم
ceph fs snapshot mirror checkpoint add cephfs /volumes/app/app1 <SNAP>
# آخرین snapshot را checkpoint می‌کند؛ بدون دانستن نام snapshot
ceph fs snapshot mirror checkpoint now cephfs /volumes/app/app1
# وضعیت checkpointها را نشان می‌دهد؛ complete یعنی به مقصد رسیده
ceph fs snapshot mirror checkpoint ls cephfs /volumes/app/app1 --format json-pretty
```

وضعیت‌ها: `created` (منتظر sync)، `complete` (به مقصد رسیده)، `failed`. حذف snapshot قبل از `complete` شدن، checkpoint را برای همیشه از بین می‌برد. به‌صورت پیش‌فرض ۱۰ checkpoint کامل آخر نگه داشته می‌شود.

---

## Disaster Recovery و Failover (دستی)

CephFS دستور promote ندارد. Failover کاملاً دستی است: توقف mirror، تأیید نقطهٔ بازیابی، هدایت کلاینت‌ها به مقصد. فقط مسیرهای `mirror add` شده در مقصد هستند.

### Failover برنامه‌ریزی‌شده (مبدأ سالم)

```bash
# ۱) اپ‌ها را متوقف کن؛ در صورت نیاز ورود کلاینت جدید را ببند
# ورود کلاینت جدید به FS مبدأ را می‌بندد؛ در failover برنامه‌ریزی‌شده برای قطع نوشتن
ceph fs set cephfs refuse_client_session true

# ۲) snapshot نهایی (برای subvolume)
# snapshot نهایی می‌سازد؛ نقطهٔ بازیابی که باید به مقصد برسد
ceph fs subvolume snapshot create cephfs app1 FINAL-$(date +%Y%m%d-%H%M) --group_name app

# ۳) صبر کن تا sync تمام شود و last_synced_snap.name همین snapshot باشد، state: idle
# وضعیت sync را چک می‌کند؛ باید idle و last_synced_snap همان snapshot نهایی باشد
ceph fs snapshot mirror status cephfs /volumes/app/app1      # یا admin socket

# ۴) فقط بعد از تأیید، mirror را متوقف کن
# mirroring را خاموش می‌کند؛ فقط بعد از تأیید sync نهایی
ceph fs snapshot mirror disable cephfs
# daemon mirror را متوقف می‌کند؛ تا دیگر چیزی به مقصد ارسال نشود
ceph orch stop cephfs-mirror
```

روی **مقصد**:

```bash
# ۵) اگر refuse_client_session ست شده بود
# محدودیت ورود کلاینت را در مقصد برمی‌دارد؛ تا کلاینت‌ها وصل شوند
ceph fs set cephfs refuse_client_session false

# ۶) mount مستقیم مسیر داده (بدون وابستگی به subvolume API)
# مسیر داده را مستقیم در مقصد mount می‌کند؛ بدون نیاز به subvolume API
ceph-fuse -n client.app1 --client_fs cephfs \
  -r /volumes/app/app1/<UUID> /mnt/app1 -o allow_other
```

برای mount در مقصد، کاربر cephx با caps روی مسیر `/volumes/app/app1/<UUID>` بساز (مثل بخش ۵).

### Failover اضطراری (مبدأ از دست رفته)

- اگر آخرین وضعیت مبدأ `idle` بود، محتوای زندهٔ مقصد معمولاً برابر آخرین snapshot sync‌شده است.
- اگر هنگام قطع، sync در حال انجام بود (`syncing`) یا مطمئن نیستی، **به محتوای زنده اعتماد نکن** (xattr `ceph.mirror.dirty_snap_id` نشانهٔ sync ناقص است). از یک snapshot سالم در `.snap/` بازیابی کن:

```bash
# snapshotهای مقصد را لیست می‌کند؛ برای انتخاب نقطهٔ بازیابی
ls /mnt/cephfs-dr/volumes/app/app1/.snap/

# بازیابی بخشی
# فقط فایل یا پوشهٔ مشخصی را از snapshot برمی‌گرداند؛ بازیابی بخشی
cp -a  <mount>/volumes/app/app1/.snap/<SNAP>/<UUID>/<path>  <mount>/volumes/app/app1/<UUID>/<path>

# بازیابی کامل (مخرب: فایل‌های اضافه در حالت زنده پاک می‌شوند)
# کل محتوا را از snapshot برمی‌گرداند؛ مخرب است و فایل‌های اضافه را پاک می‌کند
rsync -a --delete \
  <mount>/volumes/app/app1/.snap/<SNAP>/<UUID>/ \
  <mount>/volumes/app/app1/<UUID>/
```

### بعد از Failover

- فقط یک سایت قابل‌نوشتن باشد (در غیر این صورت split-brain و بدون merge خودکار).
- Failback خودکار وجود ندارد. قبل از برگرداندن نقش‌ها، همهٔ mirror daemonها را متوقف کن و دایرکتوری‌های sync‌شده را روی peer جدید پاک‌سازی کن (مطابق بخش Re-adding Peers در مستندات).
- قبل از ورود ترافیک: `ceph fs status`، `ceph health detail`، بررسی تعداد و حجم فایل‌ها و تست اپلیکیشن.
- این روال را قبل از نیاز واقعی، در تست انجام بده.

---

## حذف مسیر / peer

```bash
# mirror مسیر subvolume را متوقف می‌کند
ceph fs snapshot mirror remove cephfs /volumes/app/app1
# mirror مسیر app1-data را متوقف می‌کند
ceph fs snapshot mirror remove cephfs /app1-data
# mirror مسیر newtest را متوقف می‌کند
ceph fs snapshot mirror remove cephfs /newtest

# peerها را لیست می‌کند؛ UUID برای حذف لازم است
ceph fs snapshot mirror peer_list cephfs
# peer را حذف می‌کند؛ ارتباط با مقصد قطع می‌شود
ceph fs snapshot mirror peer_remove cephfs <peer-uuid>
# mirroring را برای FS خاموش می‌کند؛ آخرین مرحلهٔ حذف
ceph fs snapshot mirror disable cephfs
```

حذف snapshot روی مسیر معمولی:

```bash
# snapshot تست را حذف می‌کند؛ تمیزکاری
rmdir /mnt/cephfs-source/newtest/.snap/NEWTESTSNAP
```

روی Subvolume (API) در **مبدأ**:

```bash
# snapshot subvolume را در مبدأ حذف می‌کند؛ در sync بعدی از مقصد هم پاک می‌شود
ceph fs subvolume snapshot rm cephfs app1 <SNAP> --group_name app --force
```

حذف snapshot در مبدأ، در sync بعدی روی مقصد هم اعمال می‌شود (`snaps_deleted`). برای peer جدید یا peer قبلی: قبل از re-add، مطمئن شو همهٔ daemonها sync را متوقف کرده‌اند و دایرکتوری‌های قبلی را روی مقصد پاک‌سازی کن.

---

## عیب‌یابی سریع (Mirroring)

| علامت | بررسی / راهکار |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps --daemon-type cephfs-mirror` و `cephadm logs --name ...` |
| `AF_UNIX path too long` | symlink به `/tmp/mirror.asok` |
| import خطا | token، شبکه به mon مقصد، نام FS، ساعت دو کلاستر |
| `snaps_synced: 0` و `snap_map={}` | اگر مسیر mirror شامل UUID است، آن را `remove` کن و ریشهٔ subvolume (`/volumes/<group>/<subvol>`) را `add` کن. نام snapshot نباید با `_` شروع شود |
| `mkdir <UUID>/.snap/...` → Operation not permitted | طبیعی است (فرزند ریشهٔ subvolume). snapshot را روی `/volumes/<group>/<subvol>/.snap/` یا با API بساز |
| `failed` با `snapshot 'x' has invalid metadata` | روی مقصد snapshot دستی یا snap-schedule وجود دارد. آن snapshot مقصد را حذف کن و دیگر روی مقصد snapshot نساز |
| `failed` برای مسیری که هنوز وجود ندارد | مسیر اشتباه یا حذف‌شده است؛ مسیر را بساز یا `mirror remove` بزن |
| وضعیت `stale` | daemon زنده‌ای مالک مسیر نیست؛ `ceph orch ps --daemon-type cephfs-mirror` و restart |
| authentication error | `ceph auth get client.mirror_remote` روی مقصد؛ caps `rwps` |
| فایل در مقصد نیست | داخل `.snap/<name>/` را ببین؛ mirror فقط بعد از snapshot است |
| PG undersized / HEALTH_WARN | موقتاً `size 2` / `min_size 1`؛ بعد از recover دوباره تست کن |
| kernel mount secret failed | از `ceph-fuse` استفاده کن |

---

## نکات مهم

1. **مرجع داده مبدأ است.** روی مسیرهای mirrorشده در مقصد به‌صورت مستقل ننویس و snapshot نساز.
2. RPO = فاصلهٔ snapshotها + زمان sync. این DR است، نه backup؛ حذف یا خرابی داده در مبدأ در snapshot بعدی به مقصد می‌رود (snapshotهای قبلی مقصد می‌مانند).
3. `client.mirror_remote` فقط برای ارتباط بین کلاسترهاست؛ برای اپ از `client.app1` / `client.db1` استفاده کن.
4. برای Subvolume مسیر mirror = `/volumes/<group>/<subvol>`؛ مسیر mount اپ = مسیر UUID. این دو را قاطی نکن.
5. هر subvolume یک مسیر mirror جداگانه؛ مسیر والد (`/volumes`, `/volumes/<group>`) را add نکن.
6. فقط یک `cephfs-mirror` daemon.
7. ساعت دو کلاستر (NTP) و شبکه از mirror daemon به monهای مقصد (3300/6789) باید برقرار باشد.
8. Quota و سایر ویژگی‌های subvolume (clone، earmark، metadata) روی مقصد خودکار ساخته نمی‌شود؛ فقط داده و ساختار دایرکتوری.
9. بعد از هر ارتقای Ceph، یک بار تست end-to-end (مرحلهٔ ۹) را تکرار کن.

---

## خلاصه چک‌لیست عملی

```text
[ ] هر دو کلاستر HEALTH_OK (یا size تنظیم‌شده)
[ ] mirroring module + cephfs-mirror daemon (فقط ۱ عدد) روی مبدأ
[ ] client.mirror_remote (rwps) + token روی مقصد → import روی مبدأ
[ ] peer_list نشان‌دهندهٔ site-b
[ ] subvolumegroup روی مقصد ساخته شده (subvolume نه)
[ ] برای هر subvolume: mirror add روی /volumes/<group>/<subvol> (نه UUID)
[ ] snapshot با ceph fs subvolume snapshot create یا snap-schedule --subvol --group
[ ] روی مقصد هیچ snapshot دستی / snap-schedule نیست
[ ] peer status → state idle و snaps_synced >= 1
[ ] روی مقصد: .snap/<name>/<UUID>/... محتوا را تأیید کن
[ ] روال Failover در تست انجام و مستند شده
```
