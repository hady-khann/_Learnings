# راه‌اندازی CephFS Mirroring بین دو کلاستر (cephadm)

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

اگر در یکی نبود (مثلاً روی site-b):

```bash
ceph fs volume create cephfs
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

## عیب‌یابی سریع

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
