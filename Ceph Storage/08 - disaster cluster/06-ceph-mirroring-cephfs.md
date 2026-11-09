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

دسترسی کامل به کل filesystem:

```bash
ceph fs authorize cephfs client.app1 / rw
```

دسترسی فقط به یک زیردایرکتوری (پیشنهادی، اصل حداقل دسترسی):

```bash
ceph fs authorize cephfs client.app1 /data rw
```

برای اینکه کاربر بتواند snapshot هم بسازد، `rws` یا برای تغییر quota، `rwp`:

```bash
ceph fs authorize cephfs client.app1 /data rws
```

> اگر برای همین کاربر دوباره `authorize` بزنی و مسیر جدیدی بدهی، caps قبلی ممکن است جایگزین یا خطادار شود؛ قبلش `ceph auth get client.app1` را ببین. برای تغییر دسترسی یک کاربر موجود از `ceph auth caps` استفاده کن.

بررسی:

```bash
ceph auth get client.app1
```

> مسیرها داخل `ceph fs authorize` نسبت به ریشه filesystem هستند. اگر دایرکتوری `/data` هنوز وجود ندارد، بعد از اولین mount با ادمین آن را بساز (مرحله ۷).

## ۶. آماده‌سازی کلاینت

روی نود ادمین، فایل کانفیگ مینیمال و کلید را بگیر:

```bash
ceph config generate-minimal-conf > /tmp/ceph.conf
ceph auth get-key client.app1 > /tmp/app1.secret
```

این دو فایل را به سرور کلاینت ببر (مثلاً با `scp`) و روی کلاینت:

```bash
apt install -y ceph-common
mkdir -p /etc/ceph
mv /tmp/ceph.conf /etc/ceph/ceph.conf
mv /tmp/app1.secret /etc/ceph/app1.secret
chmod 600 /etc/ceph/app1.secret
```

- کلاینت باید به mon ها (پورت‌های 3300 و 6789)، به OSD ها (6800 تا 7568) و به MDS ها دسترسی شبکه داشته باشد.
- فایل `app1.secret` فقط خود کلید است، نه کل keyring.

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

### جایگزین: ceph-fuse

اگر kernel client قابل استفاده نیست:

```bash
apt install -y ceph-fuse
ceph-fuse -n client.app1 --client_fs cephfs /mnt/cephfs
```

(این روش به keyring در `/etc/ceph/ceph.client.app1.keyring` نیاز دارد: `ceph auth get client.app1 > /etc/ceph/ceph.client.app1.keyring`.)

## ۸. Quota

روی یک دایرکتوری mount شده (کاربر باید حرف `p` داشته باشد یا با admin):

```bash
apt install -y attr
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data    # 100 گیگابایت
setfattr -n ceph.quota.max_files -v 1000000 /mnt/cephfs/data
getfattr -n ceph.quota.max_bytes /mnt/cephfs/data
```

برای برداشتن quota مقدار را `0` بگذار. quota در سمت کلاینت اعمال می‌شود و سخت‌گیرانه (hard) نیست؛ کلاینت‌های مخرب یا قدیمی می‌توانند از آن عبور کنند.

## ۹. Subvolume (اختیاری)

```bash
ceph fs subvolumegroup create cephfs mygroup
ceph fs subvolume create cephfs vol1 --group_name mygroup --size 10737418240
ceph fs subvolume getpath cephfs vol1 --group_name mygroup
ceph fs subvolume ls cephfs --group_name mygroup
```

- `--size` اندازه quota به بایت است (اینجا ۱۰ گیگابایت).
- `getpath` مسیر داخل filesystem را می‌دهد (مثل `/volumes/mygroup/vol1/<uuid>`). همین مسیر را در `authorize` یا mount و در mirroring (`snapshot mirror add`) به کار ببر.
- حذف: `ceph fs subvolume rm cephfs vol1 --group_name mygroup`

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
| `mount error 13 = Permission denied` | کلید اشتباه، یا caps کاربر؛ `ceph auth get client.app1` |
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

جهت: یک‌طرفه، از `site-a` (مبدأ) به `site-b` (مقصد). مبتنی بر snapshot است و فقط snapshot ها منتقل می‌شوند. این یک DR است، نه backup.

## مقادیری که باید با مقادیر خودت جایگزین کنی

| نماد | معنی |
|---|---|
| `site-a` | کلاستر مبدأ، و `<IP-site-a>` آی‌پی نود ادمین آن |
| `site-b` | کلاستر مقصد، و `<IP-site-b>` آی‌پی نود ادمین آن |
| `cephfs` | نام filesystem |
| `/data` | دایرکتوری‌ای که mirror می‌شود |

## cephx: چه کاری لازم است؟

- کاربر daemon روی site-a را cephadm خودکار می‌سازد.
- **تنها کاربری که دستی می‌سازی** `client.mirror_remote` روی **site-b** است (مرحله ۵). کلید آن داخل token می‌رود، پس keyring را کپی نمی‌کنی.

---

## مرحله ۰: آماده‌سازی (روی نود ادمین site-a و نود ادمین site-b)

روی **هر دو**:

```bash
cephadm install ceph-common
apt install -y jq
ceph -s
ceph fsid
```

- `ceph -s` باید `HEALTH_OK` بدهد.
- `ceph fsid` را روی هر دو بنویس، تا دستورها را روی کلاستر اشتباه نزنی.
- از site-a به mon های site-b دسترسی شبکه (3300 و 6789) و ssh از site-a به site-b داشته باش.

## مرحله ۱: وجود filesystem روی هر دو کلاستر

```bash
ceph fs ls
```

اگر در یکی نبود (مثلاً روی site-b)، طبق بخش اول بساز:

```bash
ceph fs volume create cephfs --placement="2"
```

## مرحله ۲: فعال‌سازی ماژول mirroring (روی هر دو کلاستر)

```bash
ceph mgr module enable mirroring
```

## مرحله ۳: اجرای daemon (روی نود ادمین site-a)

```bash
ceph orch apply cephfs-mirror
ceph orch ps --daemon-type cephfs-mirror
```

صبر کن وضعیت به `running` برسد.

## مرحله ۴: فعال‌سازی mirroring روی filesystem (روی site-a)

```bash
ceph fs snapshot mirror enable cephfs
```

## مرحله ۵: ساخت کاربر cephx برای مقصد (روی نود ادمین site-b)

```bash
ceph fs authorize cephfs client.mirror_remote / rwps
```

خروجی یک keyring است؛ لازم نیست جایی کپی‌اش کنی. بررسی:

```bash
ceph auth get client.mirror_remote
```

## مرحله ۶: ساخت token (روی نود ادمین site-b)

```bash
ceph fs snapshot mirror peer_bootstrap create cephfs client.mirror_remote site-b > /root/cephfs-token.json
chmod 600 /root/cephfs-token.json
cat /root/cephfs-token.json
```

باید یک JSON با فیلد `token` ببینی.

## مرحله ۷: انتقال token (از site-b به site-a)

روی نود ادمین **site-b**:

```bash
scp /root/cephfs-token.json root@<IP-site-a>:/root/cephfs-token.json
```

دقت کن در CephFS جهت انتقال برعکس RBD است: token روی مقصد ساخته می‌شود و به مبدأ می‌رود.

## مرحله ۸: import کردن token (روی نود ادمین site-a)

```bash
chmod 600 /root/cephfs-token.json
TOKEN=$(jq -r .token /root/cephfs-token.json)
ceph fs snapshot mirror peer_bootstrap import cephfs "$TOKEN"
unset TOKEN
```

بررسی:

```bash
ceph fs snapshot mirror peer_list cephfs
```

باید یک peer با نام `site-b` ببینی.

## مرحله ۹: افزودن دایرکتوری (روی نود ادمین site-a)

ابتدا filesystem را mount کن و دایرکتوری را بساز:

```bash
mkdir -p /mnt/cephfs
mount -t ceph :/ /mnt/cephfs -o name=admin,fs=cephfs
mkdir -p /mnt/cephfs/data
```

سپس:

```bash
ceph fs snapshot mirror add cephfs /data
```

> اگر داده داخل subvolume است، به‌جای `/data` مسیر کامل subvolume را بده (خروجی `ceph fs subvolume getpath`).

## مرحله ۱۰: ساخت snapshot (روی site-a)

بدون snapshot چیزی منتقل نمی‌شود.

تست دستی:

```bash
echo "test" > /mnt/cephfs/data/file1.txt
mkdir /mnt/cephfs/data/.snap/snap1
```

اگر خطای مجوز گرفتی: `ceph fs set cephfs allow_new_snaps true`

خودکار (برای استفاده دائمی):

```bash
ceph mgr module enable snap_schedule
ceph fs snap-schedule add /data 1h
ceph fs snap-schedule retention add /data h 24
ceph fs snap-schedule status /data
```

## مرحله ۱۱: بررسی

روی **site-a**:

```bash
ceph fs snapshot mirror daemon status
ceph fs snapshot mirror peer_list cephfs
```

روی **site-b** (بعد از چند ثانیه تا چند دقیقه):

```bash
mkdir -p /mnt/cephfs-dr
mount -t ceph :/ /mnt/cephfs-dr -o name=admin,fs=cephfs
ls /mnt/cephfs-dr/data/.snap/
ls /mnt/cephfs-dr/data/
```

## مرحله ۱۲: پاک‌سازی (روی هر دو نود ادمین)

بعد از اینکه `peer_list` روی site-a نشان داد peer اضافه شده:

```bash
shred -u /root/cephfs-token.json
```

این دستور را روی **site-a** و **site-b** هر دو بزن.

---

## عیب‌یابی سریع (mirroring)

| علامت | کار |
|---|---|
| daemon بالا نمی‌آید | `ceph orch ps --daemon-type cephfs-mirror` و `cephadm logs --name <نام-daemon>` روی site-a |
| import خطا می‌دهد | نام filesystem، سالم‌بودن token، دسترسی شبکه به mon های site-b |
| داده منتقل نمی‌شود | snapshot ساخته‌ای؟ مسیر با `snapshot mirror add` اضافه شده؟ |
| authentication error | `ceph auth get client.mirror_remote` روی site-b |

## حذف

```bash
ceph fs snapshot mirror remove cephfs /data
ceph fs snapshot mirror peer_list cephfs
ceph fs snapshot mirror peer_remove cephfs <uuid>
ceph fs snapshot mirror disable cephfs
```

## نکات

- روی دایرکتوری‌هایی که mirror می‌شوند در site-b ننویس؛ در sync بعدی بازنویسی می‌شود.
- این mirroring یک‌طرفه است و failover/failback را باید دستی انجام بدهی.
- mirroring فقط snapshot ها را منتقل می‌کند؛ تغییرات بین دو snapshot تا snapshot بعدی روی site-b دیده نمی‌شود (RPO برابر فاصله snapshot ها به‌علاوه زمان sync است).
