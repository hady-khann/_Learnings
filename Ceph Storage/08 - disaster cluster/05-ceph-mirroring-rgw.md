# راه‌اندازی RGW Multisite بین دو کلاستر (cephadm)

در RGW به جای mirror از multisite استفاده می‌شود. به‌صورت پیش‌فرض active-active است (داده‌ها دوطرفه sync می‌شوند) و ساخت کاربر و bucket از طریق zone master انجام می‌شود. این یک DR است، نه backup (حذف آبجکت هم sync می‌شود).

## مقادیری که باید با مقادیر خودت جایگزین کنی

| نماد | معنی |
|---|---|
| `site-a` | کلاستر اول، zone master |
| `site-b` | کلاستر دوم، zone ثانویه |
| `<IP-site-a>` و `<IP-site-b>` | آی‌پی نودهای ادمین |
| `<RGW-A>` | آدرس نودی از site-a که RGW روی آن اجرا می‌شود (مثلاً `http://192.168.1.11:8080`) |
| `<RGW-B>` | همین برای site-b (مثلاً `http://192.168.2.11:8080`) |
| `myrealm`، `mygroup`، `zone-a`، `zone-b` | نام realm، zonegroup و zone ها |

## cephx: چه کاری لازم است؟

**هیچ کاربر cephx ای را دستی نمی‌سازی.** کاربر daemon های RGW را cephadm خودکار می‌سازد. تنها چیزی که بین کلاسترها می‌رود access key و secret key کاربر `sync-user` است (کلید S3، نه cephx)، و فایل `sync-user.json` برای همین است.

---

## مرحله ۰: آماده‌سازی (روی نود ادمین site-a و نود ادمین site-b)

روی **هر دو**:

```bash
apt install -y jq
cephadm shell -- ceph -s
cephadm shell -- ceph fsid
```

- `ceph -s` باید `HEALTH_OK` بدهد.
- `ceph fsid` را روی هر دو بنویس، تا دستورها را روی کلاستر اشتباه نزنی.
- در این مستند دستورهای `radosgw-admin` با `cephadm shell --` اجرا می‌شوند. متغیرهای شل (`$ACCESS`، `$SECRET`) روی هاست جایگزین می‌شوند و بعد به کانتینر می‌رسند.

## بخش الف: site-a (zone master)

همه دستورهای این بخش روی **نود ادمین site-a**.

### مرحله ۱: ساخت realm و zonegroup و zone

```bash
cephadm shell -- radosgw-admin realm create --rgw-realm=myrealm --default
cephadm shell -- radosgw-admin zonegroup create --rgw-zonegroup=mygroup --endpoints=<RGW-A> --master --default
cephadm shell -- radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-a --endpoints=<RGW-A> --master --default
```

### مرحله ۲: ساخت system user و ذخیره خروجی

```bash
cephadm shell -- radosgw-admin user create --uid=sync-user --display-name="Sync User" --system > /root/sync-user.json
chmod 600 /root/sync-user.json
```

### مرحله ۳: ثبت کلیدها در zone و commit

```bash
ACCESS=$(jq -r '.keys[0].access_key' /root/sync-user.json)
SECRET=$(jq -r '.keys[0].secret_key' /root/sync-user.json)
cephadm shell -- radosgw-admin zone modify --rgw-zone=zone-a --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin period update --commit
unset ACCESS SECRET
```

### مرحله ۴: اجرای RGW برای zone-a

```bash
cephadm shell -- ceph orch apply rgw mysite --realm=myrealm --zone=zone-a --placement=1 --port=8080
cephadm shell -- ceph orch ps --daemon-type rgw
```

صبر کن وضعیت به `running` برسد. بعد تست کن:

```bash
curl <RGW-A>
```

باید یک پاسخ XML (مثل ListAllMyBucketsResult) برگردد.

## بخش ب: انتقال فایل (از site-a به site-b)

روی نود ادمین **site-a**:

```bash
scp /root/sync-user.json root@<IP-site-b>:/root/sync-user.json
```

## بخش ج: site-b (zone ثانویه)

همه دستورهای این بخش روی **نود ادمین site-b**.

### مرحله ۵: تست دسترسی به RGW مبدأ

```bash
curl <RGW-A>
```

اگر پاسخ نیامد، شبکه یا فایروال بین دو کلاستر را درست کن؛ بدون آن مرحله بعد کار نمی‌کند.

### مرحله ۶: pull کردن realm و period

```bash
chmod 600 /root/sync-user.json
ACCESS=$(jq -r '.keys[0].access_key' /root/sync-user.json)
SECRET=$(jq -r '.keys[0].secret_key' /root/sync-user.json)

cephadm shell -- radosgw-admin realm pull --url=<RGW-A> --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin realm default --rgw-realm=myrealm
cephadm shell -- radosgw-admin period pull --url=<RGW-A> --access-key="$ACCESS" --secret="$SECRET"
```

### مرحله ۷: ساخت zone ثانویه و commit

```bash
cephadm shell -- radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-b \
  --endpoints=<RGW-B> --access-key="$ACCESS" --secret="$SECRET"
cephadm shell -- radosgw-admin period update --commit
unset ACCESS SECRET
```

### مرحله ۸: اجرای RGW برای zone-b

```bash
cephadm shell -- ceph orch apply rgw mysite --realm=myrealm --zone=zone-b --placement=1 --port=8080
cephadm shell -- ceph orch ps --daemon-type rgw
```

## بررسی

روی **هر دو** نود ادمین:

```bash
cephadm shell -- radosgw-admin sync status
```

خروجی سالم: metadata sync و data sync هر دو `caught up`.

تست عملی، روی **site-a**:

```bash
cephadm shell -- radosgw-admin user create --uid=testuser --display-name="Test User"
```

access_key و secret_key را از خروجی بردار، سپس:

```bash
aws configure       # کلیدهای testuser را وارد کن
aws --endpoint-url <RGW-A> s3 mb s3://testbucket
echo hello > file.txt
aws --endpoint-url <RGW-A> s3 cp file.txt s3://testbucket/
```

چند ثانیه بعد، روی **site-b**:

```bash
aws --endpoint-url <RGW-B> s3 ls s3://testbucket/
```

(کلیدهای `testuser` روی site-b هم کار می‌کنند، چون metadata از master sync می‌شود.)

## پاک‌سازی (روی هر دو نود ادمین)

بعد از اینکه `sync status` سالم بود:

```bash
shred -u /root/sync-user.json
```

این دستور را روی **site-a** و **site-b** هر دو بزن.

---

## عیب‌یابی سریع

| علامت | کار |
|---|---|
| `realm pull` یا `period pull` خطا | `curl <RGW-A>` از site-b، صحت کلیدها، ساعت سرورها (chrony) |
| `sync status` نشان‌دهنده behind | شبکه و `cephadm shell -- radosgw-admin sync error list` |
| خطای 403 | آیا مرحله ۳ (zone modify) و commit اجرا شده؟ |
| RGW بالا نمی‌آید | `cephadm logs --name <نام-daemon-rgw>` |

## Failover (از دست رفتن master)

روی **site-b**:

```bash
cephadm shell -- radosgw-admin zone modify --rgw-zone=zone-b --master --default
cephadm shell -- radosgw-admin period update --commit
```

برگشت master قدیمی فرآیند جدا دارد؛ قبل از production روی لب تمرین کن.

## نکته

برای محافظت در برابر حذف اشتباهی از versioning روی bucket ها استفاده کن؛ حذف آبجکت هم به zone دیگر sync می‌شود.
