# راه‌اندازی RBD Mirroring بین دو کلاستر (cephadm)

جهت: یک‌طرفه، از `site-a` (مبدأ) به `site-b` (مقصد). این یک DR است، نه backup.

## مقادیری که باید با مقادیر خودت جایگزین کنی

| نماد | معنی |
|---|---|
| `site-a` | کلاستر مبدأ، و `<IP-site-a>` آی‌پی نود ادمین آن |
| `site-b` | کلاستر مقصد، و `<IP-site-b>` آی‌پی نود ادمین آن |
| `rbd` | نام pool (روی هر دو کلاستر یکی باشد) |
| `img1` | نام image نمونه |

«نود ادمین» یعنی نودی که با cephadm بوت‌استرپ شده یا label ‏`_admin` دارد.

## cephx: چه کاری لازم است؟

در RBD mirroring **هیچ کاربر cephx ای را دستی نمی‌سازی**:

- کاربر daemon (`client.rbd-mirror.*`) را `ceph orch apply rbd-mirror` روی site-b خودکار می‌سازد.
- کاربر peer (`client.rbd-mirror-peer`) را دستور `bootstrap create` روی site-a خودکار می‌سازد.
- keyring ای بین کلاسترها کپی نمی‌شود. فقط یک فایل token منتقل می‌شود.

---

## مرحله ۰: آماده‌سازی (روی نود ادمین site-a و نود ادمین site-b)

روی **هر دو**:

```bash
cephadm install ceph-common
ceph -s
ceph fsid
```

- `ceph -s` باید `HEALTH_OK` بدهد.
- `ceph fsid` را روی هر دو بنویس، تا بعداً دستورها را روی کلاستر اشتباه نزنی.

از نود ادمین site-a به site-b دسترسی ssh و شبکه (پورت‌های 3300 و 6789 به mon های طرف مقابل) داشته باش.

## مرحله ۱: ساخت pool (روی هر دو کلاستر)

```bash
ceph osd pool create rbd
rbd pool init rbd
```

اگر pool از قبل ساخته شده، فقط `rbd pool init rbd` را بزن.

## مرحله ۲: فعال‌سازی mirroring روی pool (روی هر دو کلاستر)

```bash
rbd mirror pool enable rbd image
```

## مرحله ۳: ساخت token (روی نود ادمین site-a)

```bash
rbd mirror pool peer bootstrap create --site-name site-a rbd > /root/rbd-token
chmod 600 /root/rbd-token
```

## مرحله ۴: انتقال token (از site-a به site-b)

روی نود ادمین **site-a**:

```bash
scp /root/rbd-token root@<IP-site-b>:/root/rbd-token
```

## مرحله ۵: import کردن token (روی نود ادمین site-b)

```bash
chmod 600 /root/rbd-token
rbd mirror pool peer bootstrap import --site-name site-b --direction rx-only rbd /root/rbd-token
```

## مرحله ۶: اجرای daemon (روی نود ادمین site-b)

```bash
ceph orch apply rbd-mirror --placement=1
ceph orch ps --daemon-type rbd-mirror
```

صبر کن وضعیت daemon به `running` برسد.

## مرحله ۷: ساخت image و فعال‌سازی mirroring (روی نود ادمین site-a)

یکی از دو حالت را انتخاب کن.

**حالت snapshot-based (ساده‌تر):**

```bash
rbd create rbd/img1 --size 1G
rbd mirror image enable rbd/img1 snapshot
rbd mirror snapshot schedule add --pool rbd 5m
rbd mirror image snapshot rbd/img1
```

**حالت journal-based (RPO نزدیک به صفر، سربار بیشتر):**

```bash
rbd create rbd/img1 --size 1G --image-feature exclusive-lock,journaling
rbd mirror image enable rbd/img1 journal
```

برای image ای که از قبل وجود دارد:

```bash
rbd feature enable rbd/img1 exclusive-lock,journaling
rbd mirror image enable rbd/img1 journal
```

## مرحله ۸: بررسی

روی **site-a**:

```bash
rbd mirror pool status rbd --verbose
rbd mirror image status rbd/img1
```

روی **site-b**:

```bash
rbd mirror pool info rbd
rbd mirror pool status rbd --verbose
rbd mirror image status rbd/img1
```

وضعیت سالم روی site-b: `up+replaying`. بعد از ساخت image ممکن است چند دقیقه `up+syncing` باشد.

بررسی کاربران cephx که خودکار ساخته شده‌اند:

```bash
ceph auth ls | grep rbd-mirror
```

(روی site-b باید `client.rbd-mirror.*` و روی site-a باید `client.rbd-mirror-peer` دیده شود.)

## مرحله ۹: پاک‌سازی (روی هر دو نود ادمین)

بعد از اینکه `rbd mirror pool info rbd` روی site-b نشان داد peer اضافه شده:

```bash
shred -u /root/rbd-token
```

این دستور را روی **site-b** و **site-a** هر دو بزن.

---

## عیب‌یابی سریع

| علامت | کار |
|---|---|
| `down+unknown` | `ceph orch ps --daemon-type rbd-mirror` و `cephadm logs --name <نام-daemon>` روی site-b |
| `bootstrap import` خطا می‌دهد | pool هم‌نام روی site-b هست؟ mirroring روی pool با همان mode فعال است؟ |
| `up+error` | `rbd mirror image status rbd/img1` و فیلد description |
| ارتباط برقرار نمی‌شود | `nc -zv <mon-ip> 3300` و `nc -zv <mon-ip> 6789` بین نودها |

## Failover (در صورت نیاز)

اضطراری، روی **site-b**:

```bash
rbd mirror image promote --force rbd/img1
```

بعد از برگشت site-a، روی **site-a**:

```bash
rbd mirror image demote rbd/img1
rbd mirror image resync rbd/img1
```

(failback منظم به mirroring دوطرفه نیاز دارد؛ در import بدون `--direction rx-only` راه‌اندازی می‌شود.)

## حذف

```bash
rbd mirror image disable rbd/img1
rbd mirror pool disable rbd
```

## نکته

Mirroring همان backup نیست: حذف یا خرابی داده هم به site-b می‌رسد. برای برگشت به نقطه زمانی، snapshot های دوره‌ای یا `rbd export-diff` هم نگه دار.
