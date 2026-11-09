# CephFS از صفر + راه‌اندازی CephFS Mirroring بین دو کلاستر (cephadm)

این مستند دو بخش دارد:

- **بخش اول:** CephFS چیست و چطور از صفر روی یک کلاستر cephadm راه‌اندازی و استفاده می‌شود (مفاهیم، ساخت filesystem، کاربر، mount، subvolume، quota).
- **بخش دوم:** mirroring یک‌طرفه از `site-a` (مبدأ) به `site-b` (مقصد). مبتنی بر snapshot است و فقط snapshot ها منتقل می‌شوند. این یک DR است، نه backup.

اگر CephFS از قبل روی هر دو کلاستر کار می‌کند، مستقیم برو سراغ بخش دوم.

> **نکته مهم (تجربه عملی / Tentacle 20.2.x):**  
> Mirroring روی **دایرکتوری معمولی** با `mkdir مسیر/.snap/نام` کاملاً کار می‌کند.  
> روی **Subvolume**، هم `mkdir .snap/...` معمولاً `Operation not permitted` می‌دهد و هم snapshotهای ساخته‌شده با `ceph fs subvolume snapshot create` توسط `cephfs-mirror` به `snap_map` اضافه نمی‌شوند و sync نمی‌شوند. جزئیات در بخش «محدودیت Subvolume و Snapshot» آمده است.

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

> برای **Mirroring**، در نسخهٔ تست‌شده (Tentacle 20.2.x) استفاده از مسیر Subvolume برای ساخت snapshot قابل‌sync مشکل‌ساز است. ترجیحاً برای مسیرهای DR از دایرکتوری معمولی استفاده کن، یا محدودیت زیر را بخوان.

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

Subvolume برای ایزوله‌سازی و quota خوب است؛ برای **مسیر mirror** در Tentacle 20.2.x محدودیت دارد (بخش مربوطه را ببین).

```bash
ceph fs subvolumegroup create cephfs app
ceph fs subvolume create cephfs app1 --group_name app --size 10737418240
ceph fs subvolume getpath cephfs app1 --group_name app
```

ساخت snapshot **مدیریتی** (برای backup محلی / CSI):

```bash
ceph fs subvolume snapshot create cephfs app1 snap1 --group_name app
ceph fs subvolume snapshot ls cephfs app1 --group_name app
```

> این snapshotها روی دیسک به صورت `_snap1_<id>` دیده می‌شوند و **توسط cephfs-mirror در تست انجام‌شده sync نشدند.**

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

## محدودیت Subvolume و Snapshot (بسیار مهم)

در تست عملی روی **Ceph Tentacle 20.2.5** مشاهده شد:

| روش ساخت Snapshot | روی دایرکتوری معمولی | روی مسیر Subvolume (`/volumes/.../<UUID>`) |
|---|---|---|
| `mkdir مسیر/.snap/نام` | ✅ کار می‌کند و mirror sync می‌کند | ❌ معمولاً `Operation not permitted` (حتی با `client.admin`) |
| `ceph fs subvolume snapshot create ...` | — | ⚠️ snapshot ساخته می‌شود و در `snapshot ls` دیده می‌شود، ولی در لاگ mirror: `local snap_map={}` و `nothing to synchronize` → **sync نمی‌شود** |

علت تقریبی:

- Subvolume با xattr/مدیریت خاص، ساخت `mkdir` داخل `.snap` را محدود می‌کند.
- Snapshotهای API روی دیسک با نام داخلی `_name_<snapid>` دیده می‌شوند؛ `build_snap_map` آن‌ها را به map قابل‌انتقال اضافه نمی‌کند (`snaps_synced` صفر می‌ماند).

**نتیجه برای این مستند:**

- برای Mirroring از **مسیر دایرکتوری معمولی** استفاده کن (مثلاً `/app1-data`, `/db1-data` یا `/newtest`).
- اگر حتماً Subvolume می‌خواهی: داده را در Subvolume نگه دار، ولی برای DR یک مسیر معمولی جدا mirror کن، یا منتظر رفع/پشتیبانی بهتر در نسخه‌های بعدی باش.
- تست موفق: مسیر `/newtest` با snapshot `NEWTESTSNAP` → `snaps_synced: 1` و محتوای فایل روی secondary تأیید شد.

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

## مرحله ۵: مسیرهای قابل‌mirror (روش توصیه‌شده)

به‌جای مسیر UUIDی Subvolume، دایرکتوری معمولی بساز و همان را mirror کن:

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

اگر هنوز می‌خواهی مسیر Subvolume را add کنی (دانستن محدودیت بالا):

```bash
APP_PATH=$(ceph fs subvolume getpath cephfs app1 --group_name app)
DB_PATH=$(ceph fs subvolume getpath cephfs db1 --group_name db)
ceph fs snapshot mirror add cephfs "$APP_PATH"
ceph fs snapshot mirror add cephfs "$DB_PATH"
```

---

## مرحله ۶: ساخت Snapshot (روش سازگار با mirror)

```bash
ceph fs set cephfs allow_new_snaps true

mkdir /mnt/cephfs-source/app1-data/.snap/snap1
mkdir /mnt/cephfs-source/db1-data/.snap/snap1
ls /mnt/cephfs-source/app1-data/.snap/
ls /mnt/cephfs-source/db1-data/.snap/
```

### زمان‌بندی (اختیاری)

```bash
ceph mgr module enable snap_schedule
ceph fs snap-schedule add /app1-data 1h
ceph fs snap-schedule retention add /app1-data h 24
ceph fs snap-schedule status /app1-data
```

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

# وضعیت کلی
ceph --admin-daemon /tmp/mirror.asok fs mirror status cephfs@2

# وضعیت هر مسیر (UUID peer را از peer_list بردار)
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@2 <PEER-UUID>
```

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

اگر `snaps_synced: 0` و در لاگ `local snap_map={}` / `nothing to synchronize` دیدی، نوع snapshot یا مسیر (Subvolume) مشکل دارد.

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

ls -la /mnt/cephfs-dr/app1-data/
ls -la /mnt/cephfs-dr/app1-data/.snap/
cat /mnt/cephfs-dr/app1-data/.snap/snap1/file.txt
```

دادهٔ **داخل snapshot** را چک کن؛ فایل «جاری» بدون snapshot جدید لزوماً به‌روز نمی‌شود.

---

## مرحله ۹: تست سریع end-to-end (الگوی تأییدشده)

روی **مبدأ**:

```bash
mkdir -p /mnt/cephfs-source/newtest
echo "NEWTESTSNAP $(date)" > /mnt/cephfs-source/newtest/data.txt
mkdir /mnt/cephfs-source/newtest/.snap/NEWTESTSNAP
ceph fs snapshot mirror add cephfs /newtest
sleep 15
ceph --admin-daemon /tmp/mirror.asok fs mirror peer status cephfs@2 <PEER-UUID>
```

روی **مقصد**:

```bash
cat /mnt/cephfs-dr/newtest/.snap/NEWTESTSNAP/data.txt
```

اگر متن را دیدی، زنجیره mirror سالم است.

---

## مرحله ۱۰: پاک‌سازی token و unmount

```bash
shred -u /root/cephfs-token.json
# fusermount3 -u /mnt/cephfs-source
# fusermount3 -u /mnt/cephfs-dr
```

---

## حذف مسیر / peer

```bash
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

روی Subvolume (API):

```bash
ceph fs subvolume snapshot rm cephfs app1 snap1 --group_name app --force
```

---

## عیب‌یابی سریع (Mirroring)

| علامت | بررسی / راهکار |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps --daemon-type cephfs-mirror` و `cephadm logs --name ...` |
| `AF_UNIX path too long` | symlink به `/tmp/mirror.asok` |
| import خطا | token، شبکه به mon مقصد، نام FS، ساعت دو کلاستر |
| `snaps_synced: 0` و `snap_map={}` | snapshot با `mkdir .snap/نام` روی **دایرکتوری معمولی** بساز؛ از Subvolume API برای mirror استفاده نکن |
| `mkdir .../.snap/...` → Operation not permitted | مسیر Subvolume است؛ از مسیر معمولی استفاده کن |
| authentication error | `ceph auth get client.mirror_remote` روی مقصد؛ caps `rwps` |
| فایل در مقصد نیست | داخل `.snap/<name>/` را ببین؛ mirror فقط بعد از snapshot است |
| PG undersized / HEALTH_WARN | موقتاً `size 2` / `min_size 1`؛ بعد از recover دوباره تست کن |
| kernel mount secret failed | از `ceph-fuse` استفاده کن |

---

## نکات مهم

1. **مرجع داده مبدأ است.** روی مسیرهای mirrorشده در مقصد به‌صورت مستقل ننویس.
2. RPO = فاصله snapshotها + زمان sync.
3. `client.mirror_remote` فقط برای ارتباط بین کلاسترهاست؛ برای اپ از `client.app1` / `client.db1` استفاده کن.
4. برای DR پایدار در این نسخه، مسیرهای معمولی (`/app1-data`, `/db1-data`) را mirror کن، نه UUIDی Subvolume را.
5. Subvolume همچنان برای CSI/quota/ایزوله‌سازی روی همان کلاستر مفید است؛ فقط با `cephfs-mirror` در این نسخه جفت خوبی نیست مگر مسیر جدا برای snapshot معمولی داشته باشی.

---

## خلاصه چک‌لیست عملی

```text
[ ] هر دو کلاستر HEALTH_OK (یا size تنظیم‌شده)
[ ] mirroring module + cephfs-mirror daemon روی مبدأ
[ ] client.mirror_remote + token روی مقصد → import روی مبدأ
[ ] peer_list نشان‌دهنده site-b
[ ] مسیر معمولی ساخته و mirror add شده
[ ] mkdir مسیر/.snap/نام  (نه subvolume snapshot create برای mirror)
[ ] peer status → snaps_synced >= 1
[ ] روی مقصد: cat .../.snap/نام/فایل
```
