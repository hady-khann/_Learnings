# مرجع سریع دستورات Ceph

مرجعی برای بررسی OSDها، Poolها و ایمیج‌های RBD (بلاک دیوایس) در کلاستر Ceph.

---

## ۱. لیست OSDها

```bash
ceph osd ls
```
فقط شناسه‌ی OSDها را نمایش می‌دهد (مثل `0`، `1`، `2` ...).

برای نمایش خواناتر همراه با هاست/وضعیت/وزن:
```bash
ceph osd tree
```

برای اطلاعات دقیق‌تر هر OSD (هاست، وضعیت up/in، کلاس دیسک):
```bash
ceph osd status
```

---

## ۲. لیست Poolها

```bash
ceph osd pool ls
```
فقط نام Poolها را نمایش می‌دهد.

برای نمایش نام Poolها همراه با شناسه‌ی عددی:
```bash
ceph osd lspools
```

برای اطلاعات کامل هر Pool (تعداد PG، size/min_size، crush rule، تگ‌های application):
```bash
ceph osd pool ls detail
```

---

## ۳. لیست ایمیج‌های RBD (در همه‌ی Poolها)

ایمیج‌های RBD داخل Poolها قرار دارند، بنابراین دستوری برای «لیست همه‌ی ایمیج‌ها در یک نگاه» وجود ندارد — باید به تفکیک هر Pool لیست بگیرید. برای دیدن اینکه کدام Poolها اپلیکیشن `rbd` روی آن‌ها فعال است:

```bash
ceph osd pool ls detail | grep rbd
```

سپس ایمیج‌های هر Pool را طبق بخش ۴ لیست بگیرید.

---

## ۴. لیست ایمیج‌های یک Pool مشخص

```bash
rbd ls <pool-name>
```

مثال:
```bash
rbd ls mypool
```

برای نمایش جزئیات (سایز، فرمت، ویژگی‌ها) هر ایمیج:
```bash
rbd ls -l <pool-name>
```

برای اطلاعات یک ایمیج مشخص:
```bash
rbd info <pool-name>/<image-name>
```

---

## ۵. لیست اسنپ‌شات‌های یک ایمیج

```bash
rbd snap ls <pool-name>/<image-name>
```

مثال:
```bash
rbd snap ls mypool/myimage
```

خروجی شامل شناسه‌ی اسنپ‌شات، نام، سایز و وضعیت protection است.

---

## ۶. لیست کلون‌های یک اسنپ‌شات

کلون‌ها فقط از یک اسنپ‌شات **protect‌شده** قابل لیست‌گیری هستند. ابتدا وضعیت protection را بررسی کنید:

```bash
rbd snap ls mypool/myimage
```
(به ستون `PROTECTED` نگاه کنید؛ باید `yes` باشد.)

سپس کلون‌های فرزند را لیست بگیرید:
```bash
rbd children <pool-name>/<image-name>@<snapshot-name>
```

مثال:
```bash
rbd children mypool/myimage@snap1
```

فرمت خروجی: `<pool>/<clone-image-name>`

---

## جدول خلاصه

| کار                              | دستور                                                       |
|-----------------------------------|--------------------------------------------------------------|
| لیست OSDها                       | `ceph osd ls`                                                 |
| نمایش درختی OSDها (هاست/وضعیت)   | `ceph osd tree`                                               |
| لیست Poolها                      | `ceph osd pool ls`                                            |
| جزئیات Poolها                    | `ceph osd pool ls detail`                                     |
| لیست ایمیج‌های یک Pool           | `rbd ls <pool>`                                                |
| جزئیات یک ایمیج                  | `rbd info <pool>/<image>`                                      |
| لیست اسنپ‌شات‌های یک ایمیج       | `rbd snap ls <pool>/<image>`                                   |
| لیست کلون‌های یک اسنپ‌شات        | `rbd children <pool>/<image>@<snap>`                           |

---

*نکته: دستورات `rbd` نیازمند فعال بودن اپلیکیشن `rbd` روی Pool مقصد هستند (`ceph osd pool application enable <pool> rbd`).*