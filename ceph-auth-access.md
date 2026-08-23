# احراز هویت و مدیریت دسترسی در Ceph (CephX)

> بر اساس کلاستر Octopus (v15.2.17) دیپلوی‌شده با cephadm — سه نود (ceph-node1, ceph-node2, ceph-node3)

---

## 1. مفهوم پایه: CephX چیست؟

Ceph از یک سیستم احراز هویت به‌نام **CephX** استفاده می‌کند که مشابه Kerberos عمل می‌کند. هر «entity» (کاربر، دیمون، یا کلاینت) یک نام دارد به فرم `<type>.<id>` — مثلاً:

- `client.admin` → کاربر ادمین پیش‌فرض
- `client.hady` → یک کاربر دلخواه که خودتان می‌سازید
- `mon.`, `osd.0`, `mgr.ceph-node1.gyvxkl` → دیمون‌های خود کلاستر هم entity محسوب می‌شوند و کلید دارند

هر entity یک **secret key** دارد که در قالب فایلی به نام **keyring** ذخیره می‌شود، به‌همراه لیستی از **capabilities (caps)** که مشخص می‌کند آن entity دقیقاً به چه بخش‌هایی از کلاستر و با چه سطحی از دسترسی (read/write/execute) اجازه دسترسی دارد.

---

## 2. دیدن کاربران و دسترسی‌های فعلی

### `ceph auth ls`
لیست کامل تمام entityهای موجود در کلاستر (شامل دیمون‌ها) به‌همراه caps هرکدام.

```
client.admin
        key: AQD...==
        caps: [mds] allow *
        caps: [mgr] allow *
        caps: [mon] allow *
        caps: [osd] allow *
```

### `ceph auth get client.admin`
نمایش کامل کلید و caps یک entity خاص (خروجی به فرمت فایل keyring هم قابل ذخیره است).

```
[client.admin]
        key = AQD...==
        caps mds = "allow *"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

### `ceph auth print-key client.admin`
فقط خودِ کلید را (بدون caps) چاپ می‌کند — برای اسکریپت‌ها کاربردی است.

```
AQDx7z1oXsomeRandomBase64Key==
```

---

## 3. دادن دسترسی ادمین به یک نود جدید

وقتی می‌گوییم یک نود «دسترسی ادمین» دارد، یعنی روی آن نود دو فایل زیر موجود است و می‌توان مستقیماً با دستور `ceph` بدون نیاز به `cephadm shell` کار کرد:

- `/etc/ceph/ceph.conf`
- `/etc/ceph/ceph.client.admin.keyring`

cephadm این کار را از طریق یک **label** به‌نام `_admin` مدیریت می‌کند.

### روش ۱ — با لیبل (روش توصیه‌شده، خودکار)

```bash
ceph orch host label add ceph-node2 _admin
```

با این دستور، cephadm به‌صورت خودکار `ceph.conf` و `ceph.client.admin.keyring` را در `/etc/ceph/` روی `ceph-node2` قرار می‌دهد و آن را به‌روز نگه می‌دارد (مثلاً اگر کلید عوض شود).

> نکته: به‌صورت پیش‌فرض، لیبل `_admin` فقط روی همان نودی است که با `cephadm bootstrap` کلاستر را راه‌اندازی کرده‌اید (در اینجا `ceph-node1`).

بررسی اینکه کدام نودها لیبل ادمین دارند:
```bash
ceph orch host ls
```
```
HOST        ADDR             LABELS  STATUS
ceph-node1  ceph-node1       _admin
ceph-node2  192.168.111.102  _admin
ceph-node3  192.168.111.103
```

حذف دسترسی ادمین از یک نود:
```bash
ceph orch host label rm ceph-node2 _admin
```

### روش ۲ — کپی دستی (وقتی روش خودکار جواب نمی‌دهد یا نسخه قدیمی‌تر دارید)

روی نود بوت‌استرپ (`ceph-node1`):
```bash
scp /etc/ceph/ceph.conf ceph-node2:/etc/ceph/ceph.conf
scp /etc/ceph/ceph.client.admin.keyring ceph-node2:/etc/ceph/ceph.client.admin.keyring
```

سپس روی `ceph-node2`، دستور `ceph -s` باید بدون نیاز به `cephadm shell` کار کند.

---

## 4. ساخت کاربر جدید در Ceph

### مفهوم Capabilities (caps)
هر caps شامل یک محدوده (`mon`, `osd`, `mgr`, `mds`) و یک سطح دسترسی است:

| سطح دسترسی | معنی |
|---|---|
| `r` | فقط خواندن |
| `w` | نوشتن |
| `x` | اجرای عملیات خاص (مثل auth) |
| `allow *` | دسترسی کامل به آن محدوده |
| `profile <name>` | مجموعه‌ای از دسترسی‌های از پیش تعریف‌شده (مثل `profile rbd`) |

### ساخت کاربر جدید با `ceph auth add`
فقط entity را می‌سازد، خروجی keyring نمی‌دهد:
```bash
ceph auth add client.hady mon 'allow r' osd 'allow rwx pool=rbd-pool'
```

### ساخت کاربر و دریافت keyring هم‌زمان — `ceph auth get-or-create` (پرکاربردترین روش)
```bash
ceph auth get-or-create client.hady \
  mon 'allow r' \
  osd 'allow rwx pool=rbd-pool' \
  -o /etc/ceph/ceph.client.hady.keyring
```

خروجی فایل `ceph.client.hady.keyring`:
```
[client.hady]
        key = AQCx9z1o...==
```

اگر کاربر از قبل وجود داشته باشد، `get-or-create` فقط کلید موجود را برمی‌گرداند (تغییری در caps نمی‌دهد).

### تغییر caps یک کاربر موجود — `ceph auth caps`
```bash
ceph auth caps client.hady mon 'allow r' osd 'allow rw pool=rbd-pool'
```
⚠️ این دستور caps قبلی را کامل **جایگزین** می‌کند، نه اضافه — همیشه کل مجموعه caps مدنظر را در دستور بنویسید.

### حذف یک کاربر
```bash
ceph auth del client.hady
```

---

## 5. Keyring — فایل و مفهوم

**Keyring** فایلی متنی است که یک یا چند کلید entity را نگه می‌دارد. مسیرهای رایج:

```
/etc/ceph/ceph.client.admin.keyring   → کلید ادمین
/etc/ceph/ceph.client.hady.keyring    → کلید کاربر سفارشی
```

### ساخت دستی یک فایل keyring خالی
```bash
ceph-authtool --create-keyring /etc/ceph/ceph.client.hady.keyring
```

### افزودن یک کلید موجود به یک keyring
```bash
ceph-authtool /etc/ceph/ceph.client.hady.keyring \
  --name=client.hady \
  --add-key=AQCx9z1o...==
```

### استخراج مستقیم کلید یک کاربر به‌صورت فایل keyring (ترکیب auth get با ریدایرکت)
```bash
ceph auth get client.hady -o /etc/ceph/ceph.client.hady.keyring
```

### چند کلید در یک فایل keyring
یک فایل keyring می‌تواند چند entity را همزمان نگه دارد — دستور `ceph-authtool` با فلگ `--merge-keyring` برای ترکیب چند فایل کاربردی است.

---

## 6. دسترسی یک یوزر مشخص روی یک نود مشخص با caps محدود

این سناریوی کاملی است که معمولاً برای یک اپلیکیشن یا سرویس روی یک نود خاص لازم دارید (مثلاً کلاینتی که فقط باید به یک pool خاص RBD دسترسی داشته باشد).

### گام ۱ — ساخت کاربر با caps محدود به یک pool خاص
```bash
ceph auth get-or-create client.app1 \
  mon 'allow r' \
  osd 'allow rwx pool=app-data' \
  -o /tmp/ceph.client.app1.keyring
```

اینجا `client.app1`:
- روی `mon` فقط اجازه خواندن (`r`) دارد — یعنی می‌تواند وضعیت کلاستر را بپرسد ولی تغییری اعمال کند نه.
- روی `osd` فقط به pool با نام `app-data` دسترسی خواندن/نوشتن/اجرا دارد — نه به بقیه‌ی poolها.

### گام ۲ — انتقال فایل‌های لازم به نود مشخص (مثلاً ceph-node3)
```bash
scp /tmp/ceph.client.app1.keyring ceph-node3:/etc/ceph/
scp /etc/ceph/ceph.conf ceph-node3:/etc/ceph/
```

> نکته: از cephadm نسخه‌ی Pacific به بعد دستور `ceph orch client-keyring set client.app1 label:<label>` هم وجود دارد که این کار را خودکار می‌کند؛ در Octopus این دستور موجود نیست، پس روش دستی (scp) استفاده می‌شود.

### گام ۳ — تست دسترسی روی همان نود
روی `ceph-node3`:
```bash
ceph --name client.app1 --keyring /etc/ceph/ceph.client.app1.keyring -s
```

اگر بخواهید هر بار مجبور به نوشتن `--name` و `--keyring` نباشید، می‌توانید در `/etc/ceph/ceph.conf` بخش زیر را اضافه کنید:
```ini
[client.app1]
        keyring = /etc/ceph/ceph.client.app1.keyring
```

### گام ۴ (اختیاری) — محدودکردن دسترسی حتی بیشتر، فقط به یک namespace داخل pool
```bash
ceph auth caps client.app1 \
  mon 'allow r' \
  osd 'allow rwx pool=app-data namespace=team-a'
```

### گام ۵ (اختیاری) — دسترسی فقط-خواندنی برای مانیتورینگ/گزارش‌گیری
```bash
ceph auth get-or-create client.readonly \
  mon 'allow r' \
  osd 'allow r pool=app-data' \
  -o /tmp/ceph.client.readonly.keyring
```

---

## 7. مثال‌های caps پرکاربرد

| هدف | caps |
|---|---|
| دسترسی کامل ادمین | `mon 'allow *' osd 'allow *' mgr 'allow *' mds 'allow *'` |
| کلاینت RBD با دسترسی به یک pool خاص | `mon 'profile rbd' osd 'profile rbd pool=my-pool'` |
| کلاینت فقط-خواندنی روی یک pool | `mon 'allow r' osd 'allow r pool=my-pool'` |
| مانیتورینگ کلاستر (بدون دسترسی نوشتن) | `mon 'allow r' mgr 'allow r'` |
| دسترسی به چند pool مشخص | `osd 'allow rwx pool=pool1, allow rwx pool=pool2'` |

---

## 8. حذف کامل و پاکسازی دسترسی یک نود

اگر می‌خواهید نودی دیگر دسترسی ادمین یا کلاینت خاصی نداشته باشد:

```bash
# حذف لیبل ادمین (روش خودکار)
ceph orch host label rm ceph-node2 _admin

# حذف دستی فایل‌های keyring از خود نود
ssh ceph-node2 "rm -f /etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.app1.keyring"

# غیرفعال‌کردن کامل کاربر در سمت کلاستر (اگر دیگر لازم نیست)
ceph auth del client.app1
```

⚠️ حذف فایل keyring از روی یک نود فقط دسترسی محلی آن نود را قطع می‌کند؛ کاربر همچنان در کلاستر وجود دارد مگر با `ceph auth del` حذف شود.
