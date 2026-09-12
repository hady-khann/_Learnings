<div dir="rtl" lang="fa">

# سخت‌افزار نود OSD: CPU، RAM و شبکه

جلسهٔ آنلاین هشتم (Iran Linux House، ۲۲ نوامبر ۲۰۲۲) با اسلایدهای **Hardware** شروع می‌شود: چند OSD روی یک سرور فیزیکی جا می‌شود، چقدر RAM لازم است، و شبکهٔ Client را از شبکهٔ Cluster جدا کنید.

این بخش لاب عملی ندارد؛ فرمول‌ها و مثال‌های اسلاید را نگه دارید. Dashboard و Grafana در پوشهٔ `12 - Dashboard` هستند.

## فرمول CPU برای OSD

اسلاید می‌گوید برای هر OSD حدود **۱ GHz** ظرفیت پردازنده بگذارید:

```text
((CPU sockets * CPU cores per socket * CPU clock GHz) / No. of OSDs) >= 1
```

مثال اسلاید: یک سوکت، ۶ هسته، ۲.۵ GHz برای ۱۲ OSD:

```text
((1 * 6 * 2.5) / 12) = 1.25  >= 1
```

هر OSD حدود ۱.۲۵ GHz می‌گیرد. دو CPU نمونه روی اسلاید:

| پردازنده | هسته × کلاک | امتیاز | OSD پیشنهادی |
|---|---|---|---|
| Intel Xeon E5-2620 v4 | ۸ × ۲.۱۰ GHz | ۱۶.۸ | تا ۱۶ OSD |
| Intel Xeon E5-2680 v4 | ۱۴ × ۲.۴۰ GHz | ۳۳.۶ | تا ۳۳ OSD |

در کلاس مشخصات E5-2695 v4 هم باز شد (۱۸ هسته / ۳۶ thread، Base ۲.۱۰ GHz، TDP ۱۲۰ W). چت کلاس RAM حدود `24gig` را برای نود نمونه گفت.

این فرمول تخمین دوره است، نه حد سخت Red Hat برای هر نسل. Hyper-threading را در صورت‌کسر نگذارید؛ اسلاید فقط **cores** را می‌شمارد.

## RAM

اسلاید Memory:

- بار متوسط: **۱ GB به‌ازای هر OSD**
- از نظر کارایی: **۲ GB به‌ازای هر OSD** بهتر است (recovery و cache)
- فرض اسلاید: **یک daemon OSD برای یک دیسک فیزیکی**
- اگر چند دیسک را به یک OSD بدهید، مصرف RAM بیشتر می‌شود
- دیسک بزرگ‌تر (مثلاً ۶ TB در برابر ۴ TB) OSD را سنگین‌تر می‌کند؛ RAM نباید bottleneck شود

در recovery، OSDها object را در حافظه نگه می‌دارند. RAM کم یعنی cache کوچک و بازسازی کندتر.

## دو شبکه: Client و Cluster

اسلاید یک دیاگرام با چهار سوئیچ نشان می‌دهد:

```text
                    Clients
         ┌──────────┴──────────┐
     SWITCH#1              SWITCH#2     ← Client Network
         │  \              /  │
    Ceph Node 1 … Ceph Node N
         │  /              \  │
     SWITCH#1              SWITCH#2     ← Cluster Network
```

- **Client Network**: MON، کلاینت RBD/CephFS/RGW، Dashboard
- **Cluster Network**: replication و recovery بین OSDها (heartbeat و backfill)

هر سمت دو سوئیچ دارد تا یک سوئیچ تک‌نقطهٔ شکست نباشد. در چت کلاس از **bond** و اینکه لینک Fiber با لینک مسی فرق دارد صحبت شد؛ اسلاید خودِ bonding را پیکربندی نمی‌کند.

برای NVMe، در چت گفته شد شبکهٔ ۱۰G ممکن است برای کل دیسک‌های NVMe کم باشد؛ نسبت SSD/NVMe به OSD در فایل `02-disk-and-chassis.md` است.
</div>
