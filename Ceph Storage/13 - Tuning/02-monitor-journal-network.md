# Monitor، journal، و Jumbo Frame

ادامهٔ اسلایدهای جلسهٔ نهم: کی OSD را down/out کنید، اندازهٔ journal (دورهٔ FileStore)، و MTU ۹۰۰۰.

## mon_osd_down_out_interval

اسلاید ۵۵: چند ثانیه صبر کنید تا OSDی که جواب نمی‌دهد را down و out علامت بزنید. برای crash کوتاه یا reboot نود مفید است تا کلاستر بلافاصله rebalance نکند.

```text
default 300
mon_osd_down_out_interval = 600
```

اسلاید ۶۰۰ ثانیه (۱۰ دقیقه) را پیشنهاد می‌کند. پیش‌فرض ۳۰۰ است.

در چت گفته شد اگر OSD فقط لحظه‌ای down شود باید توان تحمل دو برابر داشته باشید؛ همان ایدهٔ این پارامتر است، نه دستور جدا.

## osd_journal_size

اسلاید ۵۶: پیش‌فرض اندازهٔ journal OSD را صفر می‌داند؛ باید خودتان بگذارید. اندازه باید حداقل دو برابر حاصلِ سرعت دیسک × حداکثر sync interval فایل‌سیستم باشد. برای journal روی SSD معمولاً journal بزرگ‌تر از ۱۰ GB می‌سازند و min/max sync interval فایل‌استور را بالا می‌برند.

```text
osd_journal_size = 20480
```

`20480` یعنی **۲۰ GB** (واحد اسلاید مگابایت است).

> **هشدار:** این پارامتر مال دوران **FileStore + journal** است. کلاسترهای BlueStore (از Luminous به بعد پیش‌فرض) WAL/DB دارند، نه `osd_journal_size`. اسلاید دوره هنوز رقم FileStore را نشان می‌دهد؛ روی Pacific/Quincy آن را در `ceph.conf` کپی نکنید مگر واقعاً FileStore دارید.

اسلاید ۵۷ در این جلسه روی صفحه نماند؛ بعد از journal مستقیم به شبکه رفتند.

## Jumbo Frame

اسلاید ۵۸: فریم اترنت با payload بیشتر از ۱۵۰۰ بایت jumbo است. روی **همه** اینترفیس‌های Client و Cluster آن را روشن کنید تا throughput بهتر شود.

باید هم روی **هاست** و هم روی **سوئیچ** یکسان باشد. اگر یکی ۱۵۰۰ و دیگری ۹۰۰۰ باشد، فریم تکه می‌شود یا drop می‌شود.

روی `eth0`:

```bash
ifconfig eth0 mtu 9000
```

در چت `fragment` و `blk` و message queue هم مطرح شد؛ اسلاید فقط MTU را می‌گوید. توپولوژی دو سوئیچ Client و دو سوئیچ Cluster در `11 - Hardware` است.

این دستور پایدار نیست؛ برای reboot باید در netplan/NetworkManager یا کانفیگ سوئیچ هم MTU ۹۰۰۰ بماند.
