# RGW از صفر + راه‌اندازی RGW Multisite بین دو کلاستر (cephadm)

این مستند سه بخش دارد:

- **بخش اول:** مفاهیم RGW از صفر (object storage، S3، realm/zonegroup/zone، multisite).
- **بخش دوم:** راه‌اندازی اولیه RGW روی یک کلاستر (single site) و تست با S3.
- **بخش سوم:** راه‌اندازی multisite بین دو کلاستر.

اگر RGW را می‌شناسی و کلاسترهای خالی داری، مستقیم برو سراغ بخش سوم.

---

# بخش اول: مفاهیم RGW

## ۱. Object Storage چیست؟

در block storage یک دیسک خام داری، در file storage درخت دایرکتوری. در **object storage** هیچ درخت واقعی‌ای نیست؛ داده‌ها به‌صورت **object** در ظرف‌هایی به نام **bucket** نگه داشته می‌شوند و از طریق HTTP (API) خوانده و نوشته می‌شوند.

| مفهوم | توضیح |
|---|---|
| **Object** | یک فایل (داده) + متادیتا + یک نام یکتا (key). مثلاً `reports/2026/a.pdf` |
| **Bucket** | ظرف objectها؛ نامش در کل سیستم یکتاست. شبیه یک دایرکتوری سطح بالا، ولی تو در تو نیست |
| **Key** | نام کامل object داخل bucket. `/` در آن فقط یک کاراکتر است و فولدر واقعی نیست (ابزارها آن را مثل فولدر نشان می‌دهند) |
| **S3** | API استاندارد Amazon برای object storage. بیشتر ابزارها (aws cli، s3cmd، rclone، Veeam، Velero، MinIO client و ...) با آن کار می‌کنند |
| **Swift** | API دیگر (OpenStack) که RGW هم پشتیبانی می‌کند؛ معمولاً لازم نیست |
| **Access key / Secret key** | نام‌کاربری و رمز S3؛ جدا از کاربران cephx |

ویژگی‌ها: آبجکت را به‌صورت جزئی ویرایش نمی‌کنی (کل آبجکت را دوباره می‌نویسی)، مقیاس‌پذیری خیلی بالا، دسترسی از طریق شبکه/HTTP، و مناسب داده‌های غیرتراکنشی (بکاپ، آرشیو، لاگ، مدیا، artifact).

## ۲. RGW چیست؟

**RGW** (RADOS Gateway، daemon به نام `radosgw`) یک سرویس HTTP است که API S3/Swift را می‌گیرد و آن را به ذخیره روی RADOS (همان OSD ها) تبدیل می‌کند. یعنی:

```
   کلاینت S3 (aws cli / اپلیکیشن)
            │  HTTP/HTTPS
            ▼
   RGW daemon (radosgw، فرانت‌اند beast)
            │  librados
            ▼
   pool های RADOS روی OSD ها
```

نکات:

- RGW **stateless** است: چند RGW را می‌توان پشت load balancer گذاشت و بدون مشکل مقیاس داد.
- در cephadm هر RGW یک کانتینر است که با `ceph orch apply rgw` می‌سازی. خود cephadm کاربر cephx آن را می‌سازد؛ نیازی نیست دستی بسازی.
- فرانت‌اند پیش‌فرض `beast` است و روی یک پورت (مثلاً 80 یا 8080) گوش می‌دهد، با امکان TLS.

## ۳. کاربر در RGW: دو دنیای جدا

| نوع | کجا استفاده می‌شود | چطور ساخته می‌شود |
|---|---|---|
| **کاربر cephx** | ارتباط RGW با کلاستر Ceph (داخلی) | خودکار توسط cephadm |
| **کاربر S3** | کلاینت‌ها از آن برای کار با bucket ها استفاده می‌کنند | `radosgw-admin user create` |

کاربر S3 با `uid`، `display-name`، و یک یا چند جفت access/secret key شناخته می‌شود. هر bucket مالک (owner) دارد و ACL و policy روی آن اعمال می‌شود. کاربر ویژه‌ای به نام **system user** (با فلگ `--system`) برای sync بین سایت‌ها در multisite به کار می‌رود.

## ۴. Pool های RGW

RGW برای کارهای مختلف pool های جدا ساخته می‌شود (نام‌ها وابسته به zone است):

| pool | محتوا |
|---|---|
| `.rgw.root` | تنظیمات realm، zonegroup و zone |
| `<zone>.rgw.control` | هماهنگی داخلی (notify) |
| `<zone>.rgw.meta` | متادیتای کاربران و bucket ها |
| `<zone>.rgw.log` | لاگ‌ها (برای multisite مهم‌اند) |
| `<zone>.rgw.buckets.index` | ایندکس آبجکت‌های هر bucket |
| `<zone>.rgw.buckets.data` | **داده واقعی** آبجکت‌ها |
| `<zone>.rgw.buckets.non-ec` | داده‌های جانبی (مثلاً multipart در حال آپلود) |

اگر بخواهی هزینه ظرفیت را کم کنی، می‌توانی `buckets.data` را روی erasure coding بگذاری. index بهتر است روی SSD/NVMe باشد.

## ۵. سلسله‌مراتب Multisite: realm → zonegroup → zone

برای اینکه RGW بتواند بین چند کلاستر هماهنگ شود، ساختار منطقی زیر تعریف می‌شود:

```
realm  (myrealm)                      ← یک فضای نام سراسری؛ کل مجموعه
 └── zonegroup  (mygroup)             ← گروهی از zone ها (شبیه یک region)؛ یکی master
      ├── zone  zone-a  (master)      ← یک یا چند RGW + pool های خودش  ← کلاستر site-a
      └── zone  zone-b                ← ...                            ← کلاستر site-b
```

| مفهوم | توضیح |
|---|---|
| **Realm** | بالاترین سطح؛ همه چیز (کاربران، bucket ها، تنظیمات) داخل یک realm مشترک است. هر realm یک **master zonegroup** دارد |
| **Zonegroup** | مجموعه‌ای از zone ها که داده بین‌شان sync می‌شود. **master zone** مشخص می‌کند متادیتا کجا تغییر می‌کند |
| **Zone** | یک کلاستر Ceph (یا بخشی از آن) به‌همراه RGW ها و pool های خودش |
| **Endpoint** | آدرس HTTP RGW هر zone که zone های دیگر برای sync با آن تماس می‌گیرند. در production بهتر است آدرس load balancer باشد |
| **Period** | یک «نسخه» از کل پیکربندی realm. هر تغییر در ساختار (zone جدید، master جدید) باید با `period update --commit` ثبت شود؛ شبیه `git commit`. بعد از commit همه zone ها از طریق `period pull` نسخه جدید را می‌گیرند. هر نسخه یک epoch دارد |

### Sync چطور کار می‌کند؟

- **Metadata sync:** ساخت/تغییر کاربر و bucket فقط روی **master zone** انجام می‌شود و به zone های دیگر کپی می‌شود (log-based).
- **Data sync:** محتوای آبجکت‌ها بین zone ها منتقل می‌شود. به‌صورت پیش‌فرض **active-active** است: هر zone می‌تواند آبجکت بنویسد و تغییرات دوطرفه sync می‌شود (در تعارض، آخرین نوشتن برنده است).
- sync **async** است؛ یعنی چند ثانیه یا بیشتر تأخیر طبیعی است (RPO غیرصفر).
- حذف آبجکت هم sync می‌شود. بنابراین **DR است، نه backup**؛ حذف اشتباهی در یک zone در zone دیگر هم اعمال می‌شود. برای محافظت از versioning استفاده کن.

### چرا نه mirror؟

RBD و CephFS روی لایه pool/snapshot کار می‌کنند و mirroring دارند. RGW لایه HTTP/متادیتای خودش را دارد و برای همین از **multisite** (در سطح S3) استفاده می‌کند.

## ۶. Load balancer، DNS و TLS

- کلاینت‌ها نباید به یک RGW مشخص وصل شوند؛ در production یک VIP/Load Balancer جلوی RGW ها بگذار (در cephadm: سرویس `ingress` با haproxy + keepalived).
- آدرس endpoint های zone در multisite ترجیحاً همان آدرس LB باشد.
- برای دسترسی virtual-host style (`bucket.s3.example.com`) باید `rgw_dns_name` را تنظیم و DNS wildcard بسازی. اگر مستقیم به IP وصل می‌شوی (مثل مثال‌های اینجا) از path-style استفاده می‌کنی و نیازی به DNS نیست.
- در production روی ارتباط کلاینت و sync حتماً TLS (HTTPS) فعال کن؛ در این مستند برای سادگی HTTP است.

## ۷. مفاهیم کاربردی دیگر

| مفهوم | توضیح |
|---|---|
| **Versioning** | هر بار نوشتن، نسخه جدیدی از آبجکت نگه می‌دارد؛ حذف فقط یک delete-marker می‌گذارد. ضد حذف اشتباهی |
| **Lifecycle** | قوانین خودکار حذف/انتقال آبجکت‌ها بعد از مدت مشخص |
| **Quota** | محدودیت اندازه و تعداد آبجکت برای کاربر یا bucket |
| **Multipart upload** | آپلود فایل بزرگ در چند قسمت |
| **Presigned URL** | لینک موقت برای دسترسی به آبجکت بدون داشتن کلید |
| **Bucket policy / ACL** | کنترل دسترسی |

---

# بخش دوم: راه‌اندازی اولیه RGW (یک کلاستر، single site)

هدف: بالا آوردن RGW روی یک کلاستر و تست با S3. اگر می‌خواهی بعداً multisite بسازی، مستقیم به بخش سوم برو و **این بخش را روی همان کلاستر اجرا نکن** (چون بخش سوم realm/zone را از ابتدا با اسم خودش می‌سازد؛ قاطی‌کردن با zone ی `default` باعث دردسر در تبدیل می‌شود).

## مرحله ۱: پیش‌نیاز

روی نود ادمین:

```bash
cephadm shell -- ceph -s
cephadm shell -- ceph orch host ls
```

`HEALTH_OK` و OSD های سالم لازم است. RGW در ساخت اولیه pool های خودش را می‌سازد، پس OSD کافی (ترجیحاً ۳ نود) لازم است.

## مرحله ۲: اجرای RGW

```bash
cephadm shell -- ceph orch apply rgw myrgw --placement="2" --port=8080
cephadm shell -- ceph orch ps --daemon-type rgw
```

- `myrgw` نام سرویس است.
- `--placement="2"` یعنی ۲ daemon (روی ۲ نود). برای نودهای مشخص: `--placement="2 node1 node2"`.
- بدون `--realm`/`--zone` یک zone پیش‌فرض (`default`) ساخته می‌شود.
- صبر کن وضعیت به `running` برسد (ممکن است ۱ تا ۲ دقیقه طول بکشد چون pool ها ساخته می‌شوند).

pool های ساخته شده:

```bash
cephadm shell -- ceph osd pool ls | grep rgw
```

## مرحله ۳: تست اتصال

```bash
curl http://<IP-نود-rgw>:8080
```

باید یک XML شبیه `ListAllMyBucketsResult` برگردد (دسترسی anonymous است، پس لیست خالی).

IP نودی که RGW روی آن است را از `ceph orch ps --daemon-type rgw` ببین.

## مرحله ۴: ساخت کاربر S3

```bash
cephadm shell -- radosgw-admin user create --uid=testuser --display-name="Test User"
```

از خروجی JSON، `access_key` و `secret_key` را بردار. بعداً دوباره دیدن کلیدها:

```bash
cephadm shell -- radosgw-admin user info --uid=testuser
```

کلید جدید یا اضافه:

```bash
cephadm shell -- radosgw-admin key create --uid=testuser --key-type=s3 --gen-access-key --gen-secret
```

## مرحله ۵: تست با aws cli

روی یک کلاینت:

```bash
apt install -y awscli
aws configure       # access/secret را وارد کن؛ region را us-east-1 بگذار؛ output پیش‌فرض
export EP=http://<IP-نود-rgw>:8080

aws --endpoint-url $EP s3 mb s3://testbucket
echo hello > file.txt
aws --endpoint-url $EP s3 cp file.txt s3://testbucket/
aws --endpoint-url $EP s3 ls s3://testbucket/
aws --endpoint-url $EP s3 cp s3://testbucket/file.txt -
```

بررسی از سمت ادمین:

```bash
cephadm shell -- radosgw-admin bucket list
cephadm shell -- radosgw-admin bucket stats --bucket=testbucket
```

## مرحله ۶: Quota (اختیاری)

```bash
cephadm shell -- radosgw-admin quota set --quota-scope=user --uid=testuser --max-size=100G --max-objects=1000000
cephadm shell -- radosgw-admin quota enable --quota-scope=user --uid=testuser
```

## مرحله ۷: Versioning (اختیاری)

```bash
aws --endpoint-url $EP s3api put-bucket-versioning --bucket testbucket --versioning-configuration Status=Enabled
aws --endpoint-url $EP s3api get-bucket-versioning --bucket testbucket
```

## مرحله ۸: دسترسی‌پذیری بالا با ingress (اختیاری)

یک فایل `rgw-ingress.yaml` بساز (آدرس VIP باید آزاد و در همان شبکه باشد):

```yaml
service_type: ingress
service_id: rgw.myrgw
placement:
  count: 2
spec:
  backend_service: rgw.myrgw
  virtual_ip: 192.168.1.100/24
  frontend_port: 80
  monitor_port: 1967
```

```bash
cephadm shell --mount rgw-ingress.yaml:/tmp/rgw-ingress.yaml -- ceph orch apply -i /tmp/rgw-ingress.yaml
```

سپس کلاینت‌ها به `http://192.168.1.100` وصل می‌شوند. (برای HTTPS در ingress باید `ssl_cert` را هم تنظیم کنی.)

## عیب‌یابی سریع (راه‌اندازی اولیه)

| علامت | کار |
|---|---|
| RGW بالا نمی‌آید | `cephadm logs --name <نام-daemon-rgw>`؛ تداخل پورت؟ pool ها ساخته شده؟ |
| `curl` پاسخ نمی‌دهد | فایروال و پورت؛ IP درست نود RGW |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch` | کلید اشتباه، ساعت کلاینت و سرور (chrony)، یا endpoint اشتباه |
| `NoSuchBucket` | نام bucket، owner، endpoint درست |
| `too many PGs` / `HEALTH_WARN` بعد از ساخت | `ceph health detail`؛ PG autoscaler معمولاً خودش اصلاح می‌کند |

## حذف (فقط روی لب)

```bash
cephadm shell -- ceph orch rm rgw.myrgw
```

حذف pool ها جداگانه و با دقت انجام می‌شود (نیاز به `mon_allow_pool_delete`).

---

# بخش سوم: راه‌اندازی RGW Multisite بین دو کلاستر (cephadm)

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
- ساعت هر دو کلاستر باید دقیق باشد (chrony)؛ اختلاف ساعت باعث خطای امضا و sync می‌شود.
- هر دو کلاستر باید **خالی از RGW** باشند (یا RGW پیش‌فرض/`default` zone نداشته باشند) و نود site-b باید به آدرس `<RGW-A>` دسترسی شبکه داشته باشد.

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
| `sync status` روی site-b می‌گوید realm/zonegroup ندارد | `realm default` و `period pull` را دوباره بررسی کن؛ `radosgw-admin period get` |
| بعد از تغییر ساختار zone ها تغییر اعمال نشد | یادت رفته `period update --commit` بزنی |

## Failover (از دست رفتن master)

روی **site-b**:

```bash
cephadm shell -- radosgw-admin zone modify --rgw-zone=zone-b --master --default
cephadm shell -- radosgw-admin period update --commit
```

برگشت master قدیمی فرآیند جدا دارد؛ قبل از production روی لب تمرین کن.

## نکات

- برای محافظت در برابر حذف اشتباهی از versioning روی bucket ها استفاده کن؛ حذف آبجکت هم به zone دیگر sync می‌شود.
- در production به‌جای IP نود، آدرس load balancer (VIP یا DNS) را به‌عنوان endpoint zone و zonegroup بده، و از HTTPS استفاده کن.
- ساخت کاربر و bucket را فقط روی master zone انجام بده؛ ساخت روی zone ثانویه تا وقتی master در دسترس است پذیرفته نمی‌شود یا تداخل ایجاد می‌کند.
- sync بین zone ها async است؛ بلافاصله بعد از نوشتن روی یک zone، خواندن از zone دیگر ممکن است آبجکت را نشان ندهد.
