# مستند راه‌اندازی Multisite برای RGW (Object Storage)

## ۱. مرور کلی

در RGW به‌جای کلمه mirror از **multisite** استفاده می‌شود. ساختار منطقی:

- **Realm**: بالاترین سطح، یک فضای نام سراسری.
- **Zonegroup**: گروهی از zone ها (قبلاً region). فقط یک zonegroup عضو **master** است.
- **Zone**: یک کلاستر Ceph (یا گروهی از RGW ها روی یک کلاستر). هر zonegroup یک zone **master** دارد.
- **Period**: نسخه‌ای از پیکربندی realm؛ با `period update --commit` اعمال می‌شود.

رفتار: به‌صورت پیش‌فرض **active-active** است؛ داده‌ها (آبجکت‌ها) بین zone ها دوطرفه sync می‌شوند. عملیات metadata (ساخت کاربر و bucket) از طریق zone master می‌گذرد.

### فرضیات

| مورد | مقدار |
|---|---|
| realm | `myrealm` |
| zonegroup | `mygroup` |
| zone اول (master، کلاستر site-a) | `zone-a` با endpoint `http://site-a-rgw:8080` |
| zone دوم (secondary، کلاستر site-b) | `zone-b` با endpoint `http://site-b-rgw:8080` |
| روش نصب | cephadm (Quincy یا جدیدتر) |

> دستورها بر اساس مستندات رسمی Ceph است. قبل از production روی لب تست کنید. اگر روی کلاستر از قبل RGW با realm و zone پیش‌فرض (`default`) اجرا می‌شود، این روند روی کلاستر تازه ساده‌تر است و برای کلاستر دارای داده باید مهاجرت جداگانه انجام شود.

---

## ۲. پیش‌نیازها

روی هر دو کلاستر:

```bash
ceph -s           # HEALTH_OK
ceph versions
```

- هر endpoint RGW باید از کلاستر طرف مقابل در دسترس باشد (تست با `curl http://site-b-rgw:8080`).
- ساعت‌ها با chrony/NTP هماهنگ باشد (اختلاف زمان باعث خطای احراز هویت S3 می‌شود).
- در محیط production از HTTPS و endpoint های پایدار (مثلاً پشت load balancer) استفاده کنید.

---

## ۳. مراحل اولیه cephx و کاربر sync

در RGW دو مفهوم احراز هویت جدا از هم هست و نباید با هم اشتباه شود:

1. **cephx**: خود daemon های RGW با یک کاربر Ceph (مثل `client.rgw.<service>.<host>.<id>`) به کلاستر وصل می‌شوند. cephadm آن را خودکار می‌سازد.
2. **System user (کلید S3)**: zone ها با هم از طریق API خودِ RGW و با access key/secret key یک **system user** ارتباط می‌گیرند. این کاربر در کلاستر Ceph (cephx) نیست و باید دستی ساخته شود.

دستورهای پایه cephx:

```bash
ceph auth ls | grep rgw
ceph auth get client.rgw.<name>
```

اگر بدون cephadm کار می‌کنید، کاربر cephx برای RGW را دستی بسازید:

```bash
ceph auth get-or-create client.rgw.zone-a.host1 \
  mon 'allow rw' osd 'allow rwx' mgr 'allow rw' \
  -o /etc/ceph/ceph.client.rgw.zone-a.host1.keyring
```

---

## ۴. مراحل راه‌اندازی

### بخش الف: کلاستر اول، zone master (site-a)

**مرحله ۱: ساخت realm**

```bash
radosgw-admin realm create --rgw-realm=myrealm --default
```

**مرحله ۲: ساخت zonegroup (master)**

```bash
radosgw-admin zonegroup create --rgw-zonegroup=mygroup \
  --endpoints=http://site-a-rgw:8080 --master --default
```

**مرحله ۳: ساخت zone (master)**

```bash
radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-a \
  --endpoints=http://site-a-rgw:8080 --master --default
```

**مرحله ۴: ساخت system user**

```bash
radosgw-admin user create --uid=sync-user --display-name="Sync User" --system
```

خروجی شامل `access_key` و `secret_key` است. آن‌ها را یادداشت کنید.

**مرحله ۵: ثبت کلیدها در zone و commit کردن period**

```bash
radosgw-admin zone modify --rgw-zone=zone-a \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
radosgw-admin period update --commit
```

**مرحله ۶: اجرای RGW برای zone-a**

```bash
ceph orch apply rgw mysite --realm=myrealm --zone=zone-a \
  --placement=1 --port=8080
ceph orch ps | grep rgw
```

### بخش ب: کلاستر دوم، zone secondary (site-b)

**مرحله ۱: pull کردن realm و period از master**

```bash
radosgw-admin realm pull --url=http://site-a-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
radosgw-admin realm default --rgw-realm=myrealm
radosgw-admin period pull --url=http://site-a-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
```

**مرحله ۲: ساخت zone ثانویه**

```bash
radosgw-admin zone create --rgw-zonegroup=mygroup --rgw-zone=zone-b \
  --endpoints=http://site-b-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>
```

**مرحله ۳: commit کردن period**

```bash
radosgw-admin period update --commit
```

**مرحله ۴: اجرای RGW برای zone-b**

```bash
ceph orch apply rgw mysite --realm=myrealm --zone=zone-b \
  --placement=1 --port=8080
```

اگر لازم شد RGW ها را restart کنید:

```bash
ceph orch restart rgw.mysite
```

---

## ۵. بررسی و تست

وضعیت sync روی هر دو کلاستر:

```bash
radosgw-admin sync status
radosgw-admin period get
radosgw-admin zonegroup get
```

خروجی سالم `sync status` چیزی شبیه این نشان می‌دهد: metadata sync و data sync هر دو `caught up`.

تست عملی:

```bash
# روی site-a: ساخت کاربر عادی
radosgw-admin user create --uid=testuser --display-name="Test User"

# ساخت bucket و آپلود با aws cli (به endpoint مبدأ)
aws --endpoint-url http://site-a-rgw:8080 s3 mb s3://testbucket
aws --endpoint-url http://site-a-rgw:8080 s3 cp file.txt s3://testbucket/

# چند ثانیه بعد، روی zone-b
aws --endpoint-url http://site-b-rgw:8080 s3 ls s3://testbucket/
radosgw-admin bucket list
```

برای aws cli از access_key و secret_key خروجی `testuser` استفاده کنید (با `aws configure`).

---

## ۶. Failover (از دست رفتن master)

اگر zone master از دسترس خارج شد و می‌خواهید zone ثانویه را master کنید (روی site-b):

```bash
radosgw-admin zone modify --rgw-zone=zone-b --master --default
radosgw-admin period update --commit
```

بعد از برگشت master قدیمی، باید آن را با period جدید sync کنید (pull دوباره) و سپس در صورت نیاز نقش‌ها را برگردانید. این فرآیند را قبل از production روی لب تمرین کنید.

---

## ۷. عیب‌یابی

| علامت | بررسی |
|---|---|
| `realm pull` یا `period pull` خطا می‌دهد | دسترسی شبکه به endpoint master، صحت access/secret، ساعت سرورها |
| `sync status` نشان‌دهنده behind | شبکه، لاگ RGW، `radosgw-admin sync error list` |
| خطای 403 در sync | کلیدهای system user در `zone modify` ثبت شده؟ آیا `period update --commit` اجرا شده؟ |
| کاربر یا bucket ساخته‌شده روی secondary دیده نمی‌شود | metadata فقط از master می‌آید؛ sync status را بررسی کنید |
| RGW بالا نمی‌آید | `ceph orch ps`، لاگ کانتینر با `cephadm logs --name rgw.<id>` |

---

## ۸. نکات مهم

- RGW multisite به‌طور پیش‌فرض **دوطرفه** است؛ بعد از ساخت zone ثانویه می‌توانید روی هر دو zone آبجکت بنویسید. اما ساخت کاربر و bucket از طریق master متادیتا را مدیریت می‌کند.
- این مکانیزم برابر با backup نیست: حذف آبجکت هم sync می‌شود. برای محافظت در برابر حذف از **versioning** و lifecycle policy استفاده کنید.
- پیکربندی realm/zonegroup/zone حساس است و هر تغییر باید با `period update --commit` اعمال شود.
- برای ترافیک بین zone ها از HTTPS استفاده کنید و کلیدهای system user را امن نگه دارید.
