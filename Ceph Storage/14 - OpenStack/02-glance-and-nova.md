<div dir="rtl" lang="fa">

# Glance روی RBD و تلاش برای `nova boot`

بعد از keyring و secret، روی `controller` بک‌اند Glance را به Pool `images` وصل کردند، یک image به نام `cirros-app1` ساختند، و با CLI قدیمی `nova` یک VM زدند. Cinder volume در این جلسه تا ته لاب نشد.

نام‌ها در یادداشت با سبک `07 - RBD` هستند. معادل ویدیو:

| ویدیو | نام یادداشت |
|---|---|
| Glance `anisa` / `cirros_anisa_image` | `cirros-app1` |

این محیط `hoodadcloud.ir` است؛ دستورها را روی کلاستر `192.168.1.x` کپی نکنید.

## ۱. `[glance_store]` در `glance-api.conf`

فایل: `/etc/glance/glance-api.conf`

در چت کلاس این بلوک آمد و همان را در nano گذاشتند:

```ini
[glance_store]
stores = rbd
default_store = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

کامنت فایل می‌گوید `default_store` از Rocky به بعد deprecated است و باید به `enabled_backends` برود؛ در این لاب هنوز همان کلید قدیمی کار کرد.

`rbd_store_chunk_size = 8` یعنی objectهای ۸ MiB (با `rbd info` هم `order 23 (8 MiB objects)` دیده شد).

## ۲. ری‌استارت سرویس

```bash
service openstack-glance-api restart
```

```text
Failed to restart openstack-glance-api.service: Unit not found
```

اسم واحد روی این دیسترو `glance-api` است:

```bash
service glance-api restart
```

## ۳. لیست imageهای از قبل موجود

```bash
openstack image list
```

اولین `openstack` گاهی در PATH نبود. وقتی کار کرد:

```text
c7d0c659-bab1-4742-99e7-fd8ad051da96  Ubuntu_22.04  active
3e6589e6-d59c-4e2b-9bbf-be6ddf6b4dd5  cirros        active
```

یک‌بار discovery به `https://hoodadcloud.ir:9292` نرسید (DNS/شبکه از نود لاب). دانشجو در چت گفت نمی‌تواند به hoodadcloud resolve کند.

دانلود Cirros از چت:

```text
http://download.cirros-cloud.net/0.6.1/cirros-0.6.1-x86_64-disk.img
```

روی دیسک controller فایل `cirros-0.6.1-x86_64-disk.img` آمد. `wget` اول با تایپ `downlaod` خراب شد.

## ۴. ساخت image — چند دستور اشتباه

CLI قدیمی Glance:

```bash
glance image-create --name cirros-app1 --is-public true \
  --disk-format=qcow2 --container-format=bare < cirros-0.6.1-x86_64-disk.img
```

وقتی `< file` را به خط بعد شکستند:

```text
bash: syntax error near unexpected token `newline'
```

بعد:

```bash
glance image
```

```text
glance: error: argument <subcommand>: invalid choice: 'image'
```

ساب‌کامند درست `image-create` یا `image-list` است.

CLI جدید OpenStack در چت:

```bash
openstack image create --file <path> --disk-format qcow2 \
  --container-format bare --public <image_name>
```

یک‌بار خودِ رشتهٔ `<image_name>` را به‌عنوان اسم تایپ کردند. `--file cirros` هم فقط وقتی کار می‌کند که فایل همین اسم را داشته باشد؛ فایل واقعی `cirros-0.6.1-x86_64-disk.img` بود.

image نهایی که ساخته شد اسمش **`cirros-app1`** است.

## ۵. تأیید روی Glance و روی RBD

```bash
glance image-list
```

```text
ID                                    Name
1389a02e-4b7f-4806-9fcd-f5fe70a4e107  cirros-app1
3e6589e6-d59c-4e2b-9bbf-be6ddf6b4dd5  cirros
c7d0c659-bab1-4742-99e7-fd8ad051da96  Ubuntu_22.04
```

روی controller با کاربر Glance:

```bash
rbd info images/1389a02e-4b7f-4806-9fcd-f5fe70a4e107 --id glance
```

```text
rbd image '1389a02e-4b7f-4806-9fcd-f5fe70a4e107':
        size 20 MiB in 3 objects
        order 23 (8 MiB objects)
        snapshot_count: 1
        id: 197a9b854f79a
        block_name_prefix: rbd_data.197a9b854f79a
        format: 2
        features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
        op_features:
        flags:
        create_timestamp: Sun Nov 27 20:37:03 2022
        access_timestamp: Sun Nov 27 20:37:03 2022
        modify_timestamp: Sun Nov 27 20:37:03 2022
```

روی خود `ceph-1` باینری `rbd` در PATH میزبان نبود:

```bash
cephadm shell
rbd ls images
```

```text
1389a02e-4b7f-4806-9fcd-f5fe70a4e107
```

یعنی Glance واقعاً روی RADOS نوشته، نه فقط در دیتابیس Glance. UUID نام Image در Pool `images` همان id رکورد Glance است؛ `rbd info` ثابت می‌کند بایت‌ها objectهای RBD هستند (chunk ۸ MiB از `rbd_store_chunk_size`).

Horizon (`hoodadcloud.ir`) image `cirros-app1` را با Disk Format QCOW2 و Container BARE نشان داد. صفحهٔ Key Pairs خالی بود.

یک‌بار `cephadm shell` روی `ceph-1` با KeyboardInterrupt قطع شد؛ traceback مال لغو دستی است، نه خطای کلاستر.

## ۶. `nova boot` تا ته نرفت

CLI `nova` deprecated است؛ ترجیح با `openstack server create` است. در لاب:

```bash
nova boot --flavor tiny --image 1389a02e-4b7f-4806-9fcd-f5fe70a4e107 vm1
```

```text
ERROR (Conflict): Multiple possible networks found, use a Network ID to be more specific.
```

بعد `--network int` و `--network internal_network`:

```text
error: unrecognized arguments: --network internal_network
```

نسخهٔ این CLI شبکه را با `--nic net-id=<uuid>` می‌گیرد، نه `--network`. VM در این جلسه ساخته نشد.

## ۷. ephemeral در برابر volume

- **Ephemeral** (Nova → Pool `vms`): دیسک با خود VM زندگی می‌کند؛ حذف اینستنس یعنی حذف دیسک.
- **Volume** (Cinder → Pool `volumes`): دیسک جداست؛ می‌توان VM را پاک کرد و volume را به VM بعدی attach کرد.

در چت: اگر فقط Nova باشد و Cinder نباشد، دیسک VM از نظر کاربر ephemeral است و با حذف اینستنس می‌رود. برای دیسک ماندگار باید Cinder به Pool `volumes` وصل شود (secret و `client.cinder` همین جلسه‌اند؛ کانفیگ `cinder.conf` روی صفحه کامل دیده نشد).
</div>
