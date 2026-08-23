
# احراز هویت و مدیریت دسترسی در Ceph — CephX

> این راهنما بر اساس **Ceph Octopus v15.2.17** و کلاستر سه‌نودی دیپلوی‌شده با `cephadm` نوشته شده است:
>
> * `ceph-node1`
> * `ceph-node2`
> * `ceph-node3`

---

## 1. CephX چیست؟

**CephX** سیستم Authentication و Authorization داخلی Ceph است.

هر Entity در Ceph یک نام با ساختار زیر دارد:

```text id="x6rj2m"
<type>.<id>
```

مثال:

```text id="3xv5wq"
client.admin
client.hady
mon.
osd.0
mgr.ceph-node1.gyvxkl
```

هم Clientها و هم Daemonهای خود Ceph دارای Identity و Secret Key هستند.

هر Entity معمولاً شامل موارد زیر است:

```text id="q0s1e7"
Entity
 ├── Secret Key
 └── Capabilities (Caps)
```

### Secret Key

کلید محرمانه‌ای است که برای Authentication استفاده می‌شود.

### Capabilities

مشخص می‌کند Entity بعد از Authentication دقیقاً چه دسترسی‌هایی دارد.

مثلاً:

```text id="z8n6q3"
mon → allow r
osd → allow rw pool=my-pool
```

یعنی کاربر می‌تواند از MON اطلاعات بخواند و روی Pool مشخص‌شده عملیات Read/Write انجام دهد.

---

# 2. مشاهده‌ی کاربران و دسترسی‌ها

## `ceph auth ls`

لیست تمام Entityهای موجود در کلاستر را نمایش می‌دهد.

```bash id="0t7qk2"
ceph auth ls
```

خروجی شامل Clientها و Daemonهای Ceph خواهد بود.

مثلاً:

```text id="c4k8v9"
client.admin

    key: AQD...==

    caps: [mds] allow *
    caps: [mgr] allow *
    caps: [mon] allow *
    caps: [osd] allow *
```

---

## `ceph auth get`

برای مشاهده‌ی یک Entity مشخص:

```bash id="e8w2n1"
ceph auth get client.admin
```

خروجی شامل Key و Caps است:

```text id="v1m5x7"
[client.admin]

    key = AQD...==

    caps mds = "allow *"
    caps mgr = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"
```

این خروجی را می‌توان به‌عنوان فایل Keyring نیز ذخیره کرد.

---

## `ceph auth print-key`

فقط Secret Key را چاپ می‌کند:

```bash id="5k8r2p"
ceph auth print-key client.admin
```

مثال:

```text id="2f4n8x"
AQDx7z1oXsomeRandomBase64Key==
```

این دستور برای Scriptها و Automation کاربردی است.

> Secret Key را مانند Password در نظر بگیر و آن را در Git، Log یا Chat عمومی قرار نده.

---

# 3. دادن دسترسی Admin به یک Node

وقتی می‌گوییم یک Node دسترسی Admin دارد، معمولاً منظور این است که فایل‌های لازم برای استفاده‌ی مستقیم از CLI Ceph روی آن Node وجود دارند:

```text id="v3k1x9"
/etc/ceph/ceph.conf
/etc/ceph/ceph.client.admin.keyring
```

در محیط `cephadm`، روش استاندارد برای مدیریت این موضوع استفاده از Label مخصوص `_admin` است.

---

## روش ۱ — استفاده از `_admin` Label

روی Node مدیریتی:

```bash id="q9d2s6"
ceph orch host label add ceph-node2 _admin
```

cephadm فایل‌های Administration موردنیاز را روی Node قرار می‌دهد و آن‌ها را مدیریت می‌کند.

برای مشاهده‌ی Labelها:

```bash id="7w5m3k"
ceph orch host ls
```

مثلاً:

```text id="j3n8v2"
HOST        ADDR             LABELS
ceph-node1  ceph-node1       _admin
ceph-node2  192.168.111.102  _admin
ceph-node3  192.168.111.103
```

برای حذف Label:

```bash id="x6m1q8"
ceph orch host label rm ceph-node2 _admin
```

> نکته: Nodeای که `cephadm bootstrap` روی آن اجرا شده معمولاً به‌صورت پیش‌فرض Label `_admin` دارد.

---

# 4. ساخت User جدید

## مفهوم Caps

Capabilities مشخص می‌کنند User در هر بخش Ceph چه سطح دسترسی دارد.

| Permission       | مفهوم                                    |
| ---------------- | ---------------------------------------- |
| `r`              | Read                                     |
| `w`              | Write                                    |
| `x`              | Execute                                  |
| `allow *`        | دسترسی کامل                              |
| `profile <name>` | مجموعه‌ای از دسترسی‌های از پیش تعریف‌شده |

Scopeهای رایج:

```text id="m8p3v6"
mon
osd
mgr
mds
```

---

## `ceph auth add`

Entity را ایجاد می‌کند و Caps را به آن اختصاص می‌دهد.

مثال:

```bash id="x2n7c5"
ceph auth add client.hady \
  mon 'allow r' \
  osd 'allow rwx pool=rbd-pool'
```

---

## `ceph auth get-or-create`

برای ساخت User و دریافت Keyring، این روش بسیار کاربردی است.

```bash id="v4k9p1"
ceph auth get-or-create client.hady \
  mon 'allow r' \
  osd 'allow rwx pool=rbd-pool' \
  -o /etc/ceph/ceph.client.hady.keyring
```

فایل ایجادشده:

```text id="s7w3n2"
[client.hady]

    key = AQCx9z1o...==
```

اگر Entity از قبل وجود داشته باشد، `get-or-create` آن را دوباره ایجاد نمی‌کند و Key موجود را برمی‌گرداند.

> نکته: برای تغییر Caps یک User موجود، از `ceph auth caps` استفاده کن.

---

# 5. تغییر Caps

برای تغییر دسترسی‌های یک Entity:

```bash id="r8q3m1"
ceph auth caps client.hady \
  mon 'allow r' \
  osd 'allow rw pool=rbd-pool'
```

⚠️ نکته‌ی مهم:

`ceph auth caps` مجموعه‌ی Caps را **جایگزین** می‌کند.

یعنی اگر قبلاً چند Capability داشتی و در دستور جدید یکی از آن‌ها را ننویسی، آن Capability حذف می‌شود.

بنابراین همیشه **کل مجموعه‌ی Caps موردنظر** را در دستور بنویس.

---

# 6. حذف User

برای حذف کامل Entity:

```bash id="n5v8k2"
ceph auth del client.hady
```

این عملیات User را از Authentication Database کلاستر حذف می‌کند.

---

# 7. Keyring

**Keyring** یک فایل متنی است که Secret Key مربوط به یک یا چند Entity را نگه می‌دارد.

مسیرهای رایج:

```text id="m7c4x9"
/etc/ceph/ceph.client.admin.keyring
/etc/ceph/ceph.client.hady.keyring
```

---

## ایجاد Keyring خالی

```bash id="q2w8n6"
ceph-authtool --create-keyring \
  /etc/ceph/ceph.client.hady.keyring
```

---

## اضافه کردن Key به Keyring

```bash id="k5m1r8"
ceph-authtool /etc/ceph/ceph.client.hady.keyring \
  --name=client.hady \
  --add-key=AQCx9z1o...==
```

---

## استخراج Entity به فایل Keyring

```bash id="f9x3p7"
ceph auth get client.hady \
  -o /etc/ceph/ceph.client.hady.keyring
```

---

## چند Entity در یک Keyring

یک فایل Keyring می‌تواند چند Entity را نگه دارد.

برای ترکیب Keyringها می‌توان از امکانات `ceph-authtool` مانند `--merge-keyring` استفاده کرد.

---

# 8. User محدود به یک Pool

یک سناریوی رایج این است که یک Application فقط به Pool خودش دسترسی داشته باشد.

مثلاً:

```text id="j6r2m9"
client.app1
      ↓
    app-data
```

و نباید به Poolهای دیگر دسترسی داشته باشد.

## مرحله ۱ — ساخت User

```bash id="p4n8x1"
ceph auth get-or-create client.app1 \
  mon 'allow r' \
  osd 'allow rwx pool=app-data' \
  -o /tmp/ceph.client.app1.keyring
```

در این مثال:

```text id="z7m3q5"
MON → Read only
OSD → Read/Write/Execute فقط روی app-data
```

---

# 9. انتقال Keyring به Node مشخص

مثلاً برای `ceph-node3`:

```bash id="b8k2w6"
scp /tmp/ceph.client.app1.keyring \
  ceph-node3:/etc/ceph/

scp /etc/ceph/ceph.conf \
  ceph-node3:/etc/ceph/
```

سپس روی `ceph-node3`:

```bash id="r4p7x3"
ceph \
  --name client.app1 \
  --keyring /etc/ceph/ceph.client.app1.keyring \
  -s
```

---

# 10. تعریف Keyring در `ceph.conf`

برای اینکه لازم نباشد هر بار `--name` و `--keyring` را وارد کنیم، می‌توان مسیر Keyring را برای Client مشخص کرد:

```ini id="c9m2v7"
[client.app1]

    keyring = /etc/ceph/ceph.client.app1.keyring
```

سپس:

```bash id="n6x4q8"
ceph --name client.app1 -s
```

---

# 11. محدود کردن دسترسی به Namespace

اگر Pool از Namespace استفاده کند، می‌توان دسترسی را حتی محدودتر کرد.

مثال:

```bash id="t3k8m5"
ceph auth caps client.app1 \
  mon 'allow r' \
  osd 'allow rwx pool=app-data namespace=team-a'
```

در این حالت Client فقط به Namespace زیر دسترسی دارد:

```text id="w7q2n9"
Pool: app-data
Namespace: team-a
```

---

# 12. User فقط-خواندنی

برای Monitoring یا Reporting می‌توان یک User Read-only ساخت:

```bash id="h4m8x1"
ceph auth get-or-create client.readonly \
  mon 'allow r' \
  osd 'allow r pool=app-data' \
  -o /tmp/ceph.client.readonly.keyring
```

این User اجازه‌ی Write روی Pool را ندارد.

---

# 13. Caps پرکاربرد

| هدف                | Caps                                                      |
| ------------------ | --------------------------------------------------------- |
| Admin کامل         | `mon 'allow *' osd 'allow *' mgr 'allow *' mds 'allow *'` |
| RBD روی یک Pool    | `mon 'profile rbd' osd 'profile rbd pool=my-pool'`        |
| Read-only روی Pool | `mon 'allow r' osd 'allow r pool=my-pool'`                |
| Monitoring         | `mon 'allow r' mgr 'allow r'`                             |
| چند Pool مشخص      | Caps مناسب برای هر Pool به‌صورت جداگانه                   |

> برای RBD، استفاده از Profileهای استاندارد Ceph معمولاً انتخاب بهتری از ساختن Caps پیچیده به‌صورت دستی است؛ البته باید با Version و نیاز واقعی Client تطبیق داده شود.

---

# 14. حذف دسترسی Admin از Node

اگر نمی‌خواهی یک Node دیگر فایل‌های Administration را دریافت کند:

```bash id="m1v7x4"
ceph orch host label rm ceph-node2 _admin
```

برای پاک‌سازی دستی Keyringها:

```bash id="q8k3n6"
ssh ceph-node2 \
  "rm -f /etc/ceph/ceph.client.admin.keyring \
         /etc/ceph/ceph.client.app1.keyring"
```

---

# 15. تفاوت حذف Keyring و حذف User

این نکته بسیار مهم است.

حذف فایل Keyring از یک Node:

```bash id="p5r9x2"
rm -f /etc/ceph/ceph.client.app1.keyring
```

فقط Credential را از **همان Node** حذف می‌کند.

اما Entity هنوز در Authentication Database کلاستر وجود دارد.

برای حذف واقعی User از کلاستر:

```bash id="v3n7m1"
ceph auth del client.app1
```

بنابراین:

```text id="k8q4w6"
حذف Keyring
     ↓
قطع دسترسی از یک Node مشخص

ceph auth del
     ↓
حذف Entity از کل کلاستر
```

---

# خلاصه‌ی معماری CephX

```text id="f2m7x9"
Client / Daemon
      │
      │ Secret Key
      ↓
   CephX
      │
      ├── Authentication
      │       ↓
      │   آیا Entity معتبر است؟
      │
      └── Authorization
              ↓
          بررسی Caps
              ↓
       ┌──────┴──────┐
       ↓             ↓
      MON           OSD
       │             │
    allow r      allow rw
                  pool=app-data
```

## اصل مهم

**Authentication** پاسخ می‌دهد:

> «تو چه کسی هستی؟»

**Authorization / Caps** پاسخ می‌دهد:

> «چه کاری اجازه داری انجام بدهی؟»

و **Keyring** محل نگه‌داری Credential موردنیاز برای Authentication است.

```text id="n9x4k2"
Entity
  ↓
Secret Key
  ↓
CephX Authentication
  ↓
Caps
  ↓
Authorization
  ↓
Allowed Operations
```
