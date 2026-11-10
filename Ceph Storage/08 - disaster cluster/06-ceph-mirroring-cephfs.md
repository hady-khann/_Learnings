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
ceph -s
ceph osd tree
ceph orch device ls
```

- `ceph -s` باید `HEALTH_OK` بدهد و OSD ها `up` و `in` باشند.
- برای production حداقل ۳ نود با OSD، و حداقل ۲ نود برای MDS (یکی active، یکی standby).
- pool های پیش‌فرض replica با `size=3` هستند؛ با کمتر از ۳ OSD مؤثر، کلاستر **degraded / undersized** می‌ماند و می‌تواند sync را مختل کند. در لب با ۴ OSD یا کمتر موقتاً:

```bash
ceph osd pool set cephfs.cephfs.meta size 2
ceph osd pool set cephfs.cephfs.data size 2
ceph osd pool set cephfs.cephfs.meta min_size 1
ceph osd pool set cephfs.cephfs.data min_size 1
```

- روی نود ادمین `ceph-common` نصب باشد:

```bash
cephadm install ceph-common
```

---

## ۳. ساخت filesystem

```bash
ceph fs volume create cephfs --placement="2"
```

- `cephfs` نام filesystem است.
- `--placement="2"` یعنی ۲ daemon MDS. برای نود مشخص: `--placement="2 node1 node2"`.
- دو pool خودکار: `cephfs.cephfs.meta` و `cephfs.cephfs.data`.

### روش دستی

```bash
ceph osd pool create cephfs_metadata
ceph osd pool create cephfs_data
ceph fs new cephfs cephfs_metadata cephfs_data
ceph orch apply mds cephfs --placement="2"
```

---

## ۴. بررسی

```bash
ceph fs ls
ceph fs status cephfs
ceph orch ps --daemon-type mds
ceph mds stat
```

```bash
ceph fs set cephfs max_mds 1
ceph fs set cephfs allow_standby_replay true
ceph fs set cephfs allow_new_snaps true
```

---

## ۵. ساخت کاربر cephx برای کلاینت

```bash
ceph fs authorize cephfs client.adminfs / rw
# یا مسیر محدود:
ceph fs authorize cephfs client.app1 /data rws
```

برای Subvolume:

```bash
ceph fs subvolume getpath cephfs app1 --group_name app
ceph fs authorize cephfs client.app1 /volumes/app/app1/<UUID> rws
```

---

## ۶. فایل‌های احراز هویت

```bash
ceph config generate-minimal-conf > /tmp/ceph.conf
ceph auth get-key client.app1 > /tmp/app1.secret
chmod 600 /tmp/app1.secret

ceph auth get client.app1 -o /tmp/ceph.client.app1.keyring
chmod 600 /tmp/ceph.client.app1.keyring
```

---

## ۷. Mount

### Kernel (گاهی در محیط cephadm با خطای secret fail می‌شود)

```bash
mkdir -p /mnt/cephfs
mount -t ceph app1@.cephfs=/ /mnt/cephfs -o secretfile=/etc/ceph/app1.secret
```

اگر دیدی:

```text
adding ceph secret key to kernel failed: Invalid argument
```

از **ceph-fuse** استفاده کن.

### Ceph-FUSE (پیشنهادی در تست / وقتی kernel مشکل دارد)

```bash
apt install -y ceph-fuse
mkdir -p /mnt/cephfs
ceph-fuse -n client.app1 -k /etc/ceph/ceph.client.app1.keyring \
  --client_fs cephfs /mnt/cephfs
```

با root محدود (`-r`):

```bash
ceph-fuse -n client.app1 -k /path/to/keyring --client_fs cephfs \
  -r /volumes/app/app1/<UUID> /mnt/cephfs/app/app1 -o allow_other
```

---

## ۸. Quota

```bash
apt install -y attr
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data
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
ceph fs subvolumegroup create cephfs app
ceph fs subvolume create cephfs app1 --group_name app --size 10737418240
ceph fs subvolume getpath cephfs app1 --group_name app     # → /volumes/app/app1/<UUID>  (مسیر داده)
ceph fs subvolume info cephfs app1 --group_name app
```

ساخت snapshot (محلی / CSI / **و قابل mirror**، وقتی ریشهٔ subvolume mirror شده باشد):

```bash
ceph fs subvolume snapshot create cephfs app1 snap1 --group_name app
ceph fs subvolume snapshot ls cephfs app1 --group_name app
```

> این snapshot در `/volumes/app/app1/.snap/snap1` ساخته می‌شود. اگر از داخل مسیر UUID به `.snap` نگاه کنی، همان snapshot با نام `_snap1_<ino>` (snapshot «ارثی» از والد) دیده می‌شود و `cephfs-mirror` عمداً نام‌های شروع‌شده با `_` را نادیده می‌گیرد. برای همین، mirror روی مسیر UUID هیچ‌چیز sync نمی‌کند.

---

## ۱۰. Snapshot روی دایرکتوری معمولی (روش سازگار با Mirroring)

```bash
ceph fs set cephfs allow_new_snaps true
mkdir -p /mnt/cephfs-source/data
mount ...   # یا ceph-fuse با admin

mkdir /mnt/cephfs-source/data/.snap/snap-test
ls /mnt/cephfs-source/data/.snap/
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
cephadm install ceph-common
apt install -y jq attr ceph-fuse
ceph -s
ceph fsid
ceph fs ls
```

- نام FS در هر دو طرف یکسان باشد (`cephfs`).
- از مبدأ به monهای مقصد (3300 / 6789) دسترسی شبکه باشد.
- هر دو کلاستر ترجیحاً `HEALTH_OK` باشند (undersized زیاد می‌تواند نوشتن snapshot روی مقصد را مختل کند).

---

## مرحله ۱: فعال‌سازی ماژول و daemon (مبدأ)

```bash
ceph mgr module enable mirroring
ceph orch apply cephfs-mirror
ceph orch ps --daemon-type cephfs-mirror
```

صبر کن تا `running` شود.

روی **هر دو** کلاستر (در صورت نیاز ماژول):

```bash
ceph mgr module enable mirroring
```

---

## مرحله ۲: فعال‌سازی mirroring روی FS (مبدأ)

```bash
ceph fs snapshot mirror enable cephfs
```

---

## مرحله ۳: کاربر و token روی مقصد (site-b)

```bash
ceph fs authorize cephfs client.mirror_remote / rwps
ceph auth get client.mirror_remote

ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json
chmod 600 /root/cephfs-token.json
jq . /root/cephfs-token.json
```

انتقال به مبدأ:

```bash
scp /root/cephfs-token.json root@<IP-site-a>:/root/cephfs-token.json
```

---

## مرحله ۴: Import peer (مبدأ)

```bash
chmod 600 /root/cephfs-token.json
TOKEN=$(jq -r .token /root/cephfs-token.json)
ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
unset TOKEN

ceph fs snapshot mirror peer_list cephfs
```

باید peer با `site-b` دیده شود.

---

## مرحله ۵: افزودن مسیرها برای mirror (روی مبدأ)

### الف) Subvolume (روش استاندارد برای محیط عملیاتی شما)

```bash
# نام group و subvolume را عوض کن
FS=cephfs
GROUP=app
SUBVOL=app1

# بررسی: مسیر داده (فقط برای اطلاع) و ریشهٔ subvolume
ceph fs subvolume getpath $FS $SUBVOL --group_name $GROUP
# خروجی: /volumes/app/app1/<UUID>   ← این را mirror نکن

# مسیر صحیح mirror = ریشهٔ subvolume = دو سطح بالاتر از UUID
SV_ROOT=$(dirname "$(ceph fs subvolume getpath $FS $SUBVOL --group_name $GROUP)")
echo "$SV_ROOT"            # /volumes/app/app1

ceph fs snapshot mirror add $FS "$SV_ROOT"
ceph fs snapshot mirror ls $FS
```

برای subvolume در group پیش‌فرض، مسیر `/volumes/_nogroup/<subvol>` است (بدون `--group_name`).

برای چند subvolume، همین کار را برای هرکدام تکرار کن:

```bash
for SV in "app:app1" "db:db1"; do
  G=${SV%%:*}; N=${SV##*:}
  ROOT=$(dirname "$(ceph fs subvolume getpath cephfs $N --group_name $G)")
  ceph fs snapshot mirror add cephfs "$ROOT"
done
ceph fs snapshot mirror ls cephfs
```

> `dirname` روی خروجی `getpath` تا وقتی subvolume نسخهٔ v2 است درست کار می‌کند (پیش‌فرض در Ceph مدرن). با `ceph fs subvolume info` فیلد `path` و `features` را می‌توانی چک کنی.

### ب) دایرکتوری معمولی (برای داده‌ای که subvolume نیست)

```bash
# روی مبدأ – mount ادمین با FUSE
mkdir -p /mnt/cephfs-source
ceph-fuse /mnt/cephfs-source -n client.admin --client_fs cephfs -o allow_other

mkdir -p /mnt/cephfs-source/app1-data /mnt/cephfs-source/db1-data
echo "app data" > /mnt/cephfs-source/app1-data/file.txt
echo "db data"  > /mnt/cephfs-source/db1-data/file.txt

ceph fs snapshot mirror add cephfs /app1-data
ceph fs snapshot mirror add cephfs /db1-data
ceph fs snapshot mirror ls cephfs
```

---

## مرحله ۶: ساخت Snapshot (روی مبدأ)

Mirror فقط snapshotهای **با نام معمولی** (بدون پیشوند `_`) را منتقل می‌کند. نام snapshot باید در مقصد یکتا باشد؛ نام ثابت و تکراری استفاده نکن (مثلاً با تاریخ بساز).

```bash
ceph fs set cephfs allow_new_snaps true
```

### الف) Subvolume: روش A، با API (توصیه‌شده برای دستی و اسکریپت)

```bash
SNAP=snap-$(date +%Y%m%d-%H%M%S)
ceph fs subvolume snapshot create cephfs app1 $SNAP --group_name app
ceph fs subvolume snapshot ls cephfs app1 --group_name app
```

### ب) Subvolume: روش B، با mkdir روی ریشهٔ subvolume

```bash
# mount ادمین از ریشهٔ FS
mkdir /mnt/cephfs-source/volumes/app/app1/.snap/snap-$(date +%Y%m%d-%H%M%S)
ls /mnt/cephfs-source/volumes/app/app1/.snap/
```

> این همان چیزی است که API انجام می‌دهد. اگر از داخل UUID بزنی (`.../app1/<UUID>/.snap/...`) `EPERM` می‌گیری؛ طبیعی است.

### پ) Subvolume: زمان‌بندی خودکار (توصیه‌شده برای production)

```bash
ceph mgr module enable snap_schedule
ceph fs snap-schedule add / 1h --fs cephfs --subvol app1 --group app
ceph fs snap-schedule retention add / h 24 --fs cephfs --subvol app1 --group app
ceph fs snap-schedule status / --fs cephfs --subvol app1 --group app
```

- اسم snapshotها `scheduled-<timestamp>` می‌شود و mirror آن‌ها را عادی می‌بیند.
- Retention مبدأ را طوری بگذار که snapshot هنوز-sync‌نشده حذف نشود. اگر snapshot قبل از sync پاک شود، فقط snapshotهای باقی‌مانده منتقل می‌شوند (RPO بدتر می‌شود).
- روی **مقصد** هیچ snap-schedule نگذار.

### ت) دایرکتوری معمولی

```bash
mkdir /mnt/cephfs-source/app1-data/.snap/snap-$(date +%Y%m%d-%H%M%S)
ls /mnt/cephfs-source/app1-data/.snap/

ceph fs snap-schedule add /app1-data 1h
ceph fs snap-schedule retention add /app1-data h 24
ceph fs snap-schedule status /app1-data
```

### ث) Snapshot هماهنگ برای چند کلاینت (اختیاری، Squid به بعد)

Snapshot در CephFS بین چند کلاینتِ همزمان consistency قوی ندارد. برای دیتابیس‌ها یا اپ‌های چندکلاینتی، قبل از snapshot، subvolume را quiesce کن:

```bash
ceph fs quiesce cephfs --set-id snapset1 app/app1 --timeout 30 --expiration 30 --await
ceph fs subvolume snapshot create cephfs app1 snap-$(date +%Y%m%d-%H%M%S) --group_name app
ceph fs quiesce cephfs --set-id snapset1 --release --await
```

مقادیر timeout و expiration را با نیاز خودت تنظیم کن. این مرحله فقط برای اپ‌هایی لازم است که consistency بین کلاینت‌ها مهم است.

---

## مرحله ۷: بررسی وضعیت mirror (مبدأ)

### peer list و daemon

```bash
ceph fs snapshot mirror daemon status
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror ls cephfs
```

### وضعیت دقیق از admin socket

مسیر asok در cephadm طولانی است و ممکن است خطای `AF_UNIX path too long` بدهد. همیشه symlink کوتاه بساز:

```bash
for a in $(find /var/run/ceph -name '*cephfs-mirror*.asok'); do
  if ceph --admin-daemon "$a" help 2>/dev/null | grep -q 'fs mirror peer status'; then
    ln -sf "$a" /tmp/mirror.asok
    echo "ASOK=$a"
    break
  fi
done

# عدد بعد از @ همان fscid است (در مثال 2). از اینجا بگیر:
ceph fs get cephfs | grep fscid

# وضعیت کلی
ceph --admin-daemon /tmp/mirror.asok fs mirror status cephfs@<FSCID>

# وضعیت هر مسیر (UUID peer را از peer_list بردار)
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

> در نسخه‌های جدید، خروجی `peer status` داخل `metrics/<مسیر>/peer/<uuid>` تو در تو است؛ در نسخه‌های قدیمی‌تر، مسیرها مستقیم کلید اصلی‌اند. معنی فیلدها یکسان است.

### وضعیت از طریق mgr (بدون admin socket)

اگر نسخهٔ شما این دستور را دارد (در مستندات `latest` آمده؛ با `ceph fs snapshot mirror -h` چک کن)، از هر نود ادمین کار می‌کند:

```bash
ceph fs snapshot mirror status cephfs
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
cephadm logs --name $(ceph orch ps --daemon-type cephfs-mirror -f json | jq -r '.[0].daemon_name') 2>&1 | tail -100

# افزایش debug در صورت نیاز
ceph config set client.cephfs-mirror debug_cephfs_mirror 20
ceph orch daemon restart <cephfs-mirror-daemon-name>
```

عبارت‌های مفید در لاگ: `build_snap_map`, `do_sync_snaps`, `synchronizing`, `nothing to synchronize`, `connected to cluster`.

---

## مرحله ۸: بررسی روی مقصد (site-b)

```bash
mkdir -p /mnt/cephfs-dr
ceph-fuse /mnt/cephfs-dr -n client.admin --client_fs cephfs -o allow_other

# Subvolume (مسیر در مقصد همان مسیر مبدأ است)
ls -la /mnt/cephfs-dr/volumes/app/app1/
ls -la /mnt/cephfs-dr/volumes/app/app1/.snap/
ls -la /mnt/cephfs-dr/volumes/app/app1/.snap/<SNAP>/<UUID>/

# دایرکتوری معمولی
ls -la /mnt/cephfs-dr/app1-data/.snap/
cat /mnt/cephfs-dr/app1-data/.snap/<SNAP>/file.txt
```

در مقصد برای هر snapshot sync‌شده باید یک snapshot هم‌نام وجود داشته باشد. نام snapshot آخر باید با `last_synced_snap.name` در مبدأ یکی باشد.

دادهٔ **داخل snapshot** را چک کن؛ فایل «جاری» بدون snapshot جدید لزوماً به‌روز نمی‌شود.

---

## مرحله ۹: تست سریع end-to-end

### تست Subvolume (روی مبدأ)

```bash
ceph fs subvolumegroup create cephfs testgrp
ceph fs subvolume create cephfs testsv --group_name testgrp --size 1073741824

UUID_PATH=$(ceph fs subvolume getpath cephfs testsv --group_name testgrp)
SV_ROOT=$(dirname "$UUID_PATH")
echo "$UUID_PATH  →  $SV_ROOT"

# نوشتن داده داخل مسیر UUID
echo "SVTEST $(date)" > /mnt/cephfs-source${UUID_PATH}/data.txt

# ساخت snapshot و اضافه کردن مسیر
ceph fs subvolume snapshot create cephfs testsv SVTEST1 --group_name testgrp
ceph fs snapshot mirror add cephfs "$SV_ROOT"

sleep 20
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

روی **مقصد**:

```bash
UUID_DIR=$(ls /mnt/cephfs-dr/volumes/testgrp/testsv/.snap/SVTEST1/)
cat /mnt/cephfs-dr/volumes/testgrp/testsv/.snap/SVTEST1/$UUID_DIR/data.txt
```

### تست دایرکتوری معمولی (الگوی قبلی)

روی **مبدأ**:

```bash
mkdir -p /mnt/cephfs-source/newtest
echo "NEWTESTSNAP $(date)" > /mnt/cephfs-source/newtest/data.txt
mkdir /mnt/cephfs-source/newtest/.snap/NEWTESTSNAP
ceph fs snapshot mirror add cephfs /newtest
sleep 15
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@<FSCID> <PEER-UUID>
```

روی **مقصد**:

```bash
cat /mnt/cephfs-dr/newtest/.snap/NEWTESTSNAP/data.txt
```

اگر متن را دیدی، زنجیرهٔ mirror سالم است.

---

## مرحله ۱۰: پاک‌سازی token و unmount

```bash
shred -u /root/cephfs-token.json
# fusermount3 -u /mnt/cephfs-source
# fusermount3 -u /mnt/cephfs-dr
```

---

## Checkpoint (اختیاری، برای اطمینان از رسیدن snapshot مهم)

در مستندات `latest` آمده؛ با `ceph fs snapshot mirror -h` وجود آن را در نسخهٔ خودت چک کن.

```bash
ceph fs snapshot mirror checkpoint add cephfs /volumes/app/app1 <SNAP>
ceph fs snapshot mirror checkpoint now cephfs /volumes/app/app1
ceph fs snapshot mirror checkpoint ls cephfs /volumes/app/app1 --format json-pretty
```

وضعیت‌ها: `created` (منتظر sync)، `complete` (به مقصد رسیده)، `failed`. حذف snapshot قبل از `complete` شدن، checkpoint را برای همیشه از بین می‌برد. به‌صورت پیش‌فرض ۱۰ checkpoint کامل آخر نگه داشته می‌شود.

---

## Disaster Recovery و Failover (دستی)

CephFS دستور promote ندارد. Failover کاملاً دستی است: توقف mirror، تأیید نقطهٔ بازیابی، هدایت کلاینت‌ها به مقصد. فقط مسیرهای `mirror add` شده در مقصد هستند.

### Failover برنامه‌ریزی‌شده (مبدأ سالم)

```bash
# ۱) اپ‌ها را متوقف کن؛ در صورت نیاز ورود کلاینت جدید را ببند
ceph fs set cephfs refuse_client_session true

# ۲) snapshot نهایی (برای subvolume)
ceph fs subvolume snapshot create cephfs app1 FINAL-$(date +%Y%m%d-%H%M) --group_name app

# ۳) صبر کن تا sync تمام شود و last_synced_snap.name همین snapshot باشد، state: idle
ceph fs snapshot mirror status cephfs /volumes/app/app1      # یا admin socket

# ۴) فقط بعد از تأیید، mirror را متوقف کن
ceph fs snapshot mirror disable cephfs
ceph orch stop cephfs-mirror
```

روی **مقصد**:

```bash
# ۵) اگر refuse_client_session ست شده بود
ceph fs set cephfs refuse_client_session false

# ۶) mount مستقیم مسیر داده (بدون وابستگی به subvolume API)
ceph-fuse -n client.app1 --client_fs cephfs \
  -r /volumes/app/app1/<UUID> /mnt/app1 -o allow_other
```

برای mount در مقصد، کاربر cephx با caps روی مسیر `/volumes/app/app1/<UUID>` بساز (مثل بخش ۵).

### Failover اضطراری (مبدأ از دست رفته)

- اگر آخرین وضعیت مبدأ `idle` بود، محتوای زندهٔ مقصد معمولاً برابر آخرین snapshot sync‌شده است.
- اگر هنگام قطع، sync در حال انجام بود (`syncing`) یا مطمئن نیستی، **به محتوای زنده اعتماد نکن** (xattr `ceph.mirror.dirty_snap_id` نشانهٔ sync ناقص است). از یک snapshot سالم در `.snap/` بازیابی کن:

```bash
ls /mnt/cephfs-dr/volumes/app/app1/.snap/

# بازیابی بخشی
cp -a  <mount>/volumes/app/app1/.snap/<SNAP>/<UUID>/<path>  <mount>/volumes/app/app1/<UUID>/<path>

# بازیابی کامل (مخرب: فایل‌های اضافه در حالت زنده پاک می‌شوند)
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
ceph fs snapshot mirror remove cephfs /volumes/app/app1
ceph fs snapshot mirror remove cephfs /app1-data
ceph fs snapshot mirror remove cephfs /newtest

ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror peer_remove cephfs <peer-uuid>
ceph fs snapshot mirror disable cephfs
```

حذف snapshot روی مسیر معمولی:

```bash
rmdir /mnt/cephfs-source/newtest/.snap/NEWTESTSNAP
```

روی Subvolume (API) در **مبدأ**:

```bash
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
