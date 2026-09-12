# Ceph Storage

یادداشت‌های دورهٔ **Ceph Administration** (Iran Linux House) از فایل‌های `Ceph/1.mkv` تا `Ceph/7.mkv`.

شمارهٔ پوشه از **موضوع جلسه** می‌آید، نه از اسم فایل ویدیو. مثلاً `4.mkv` پوشهٔ `08` است، نه `04`. یک ویدیو ممکن است چند پوشه داشته باشد.

زبان یادداشت‌ها ترکیبی است: توضیح فارسی، دستور و مقدارها انگلیسی.

ویدیوها در `../Ceph/` هستند و اینجا commit نمی‌شوند.

## ویدیو → پوشه

| ویدیو | جلسه (روی اسلاید) | پوشه‌ها |
|---|---|---|
| `1.mkv` | ابتدای دوره (ترتیب موضوع) | [`01 - Concepts`](01%20-%20Concepts) · [`02 - Install`](02%20-%20Install) · [`03 - OSD`](03%20-%20OSD) |
| `2.mkv` | ادامهٔ لاب (ترتیب موضوع) | [`04 - Commands`](04%20-%20Commands) · [`05 - Cephx - Authentication`](05%20-%20Cephx%20-%20Authentication) |
| `3.mkv` | Pool و بلوک (ترتیب موضوع) | [`06 - Pool`](06%20-%20Pool) · [`07 - RBD`](07%20-%20RBD) |
| `4.mkv` | پنجم | [`08 - disaster cluster`](08%20-%20disaster%20cluster) |
| `5.mkv` | ششم (۱۵ نوامبر ۲۰۲۲) | [`09 - CephFS`](09%20-%20CephFS) · [`10 - RGW`](10%20-%20RGW) |
| `6.mkv` | هشتم (۲۲ نوامبر ۲۰۲۲) | [`11 - Hardware`](11%20-%20Hardware) · [`12 - Dashboard`](12%20-%20Dashboard) |
| `7.mkv` | نهم (۲۷ نوامبر ۲۰۲۲) | [`13 - Tuning`](13%20-%20Tuning) · [`14 - OpenStack`](14%20-%20OpenStack) · [`15 - HAProxy`](15%20-%20HAProxy) |

جلسهٔ هفتم فایل `.mkv` جدا در `Ceph/` ندارد.

پوشه‌های `08` تا `15` از روی فریم ویدیو نوشته شده‌اند. پوشه‌های `01` تا `07` موضوع همان سه ویدیوی اول دوره هستند؛ تقسیم `1`/`2`/`3` روی چند پوشه مثل ویدیوهای بعدی است (یک فایل، چند موضوع).

## پوشه → ویدیو

| پوشه | ویدیو |
|---|---|
| `01 - Concepts` | `1.mkv` |
| `02 - Install` | `1.mkv` |
| `03 - OSD` | `1.mkv` |
| `04 - Commands` | `2.mkv` |
| `05 - Cephx - Authentication` | `2.mkv` |
| `06 - Pool` | `3.mkv` |
| `07 - RBD` | `3.mkv` |
| `08 - disaster cluster` | `4.mkv` |
| `09 - CephFS` | `5.mkv` |
| `10 - RGW` | `5.mkv` |
| `11 - Hardware` | `6.mkv` |
| `12 - Dashboard` | `6.mkv` |
| `13 - Tuning` | `7.mkv` |
| `14 - OpenStack` | `7.mkv` |
| `15 - HAProxy` | `7.mkv` |

## فهرست فایل‌ها

- **01 - Concepts** — daemonها، pool، نوع ذخیره‌سازی، CRUSH
- **02 - Install** — `/etc/hosts`، Docker، time server، cephadm
- **03 - OSD** — اضافه کردن دیسک به OSD
- **04 - Commands** — دستورهای پایه (`ceph -s` و بقیه)
- **05 - Cephx - Authentication** — کاربر و capability
- **06 - Pool** — CLI و Ansible
- **07 - RBD** — ساخت/mount، resize، snapshot، clone، Ceph-CSI
- **08 - disaster cluster** — RBD mirror بین کلاستر Active و Passive
- **09 - CephFS** — مفهوم و kernel mount
- **10 - RGW** — Ansible، user/subuser، Swift
- **11 - Hardware** — CPU / RAM / شبکه و دیسک
- **12 - Dashboard** — ماژول dashboard و Grafana
- **13 - Tuning** — cache، PG، journal، jumbo frame
- **14 - OpenStack** — Cinder/Glance/libvirt روی hoodadcloud
- **15 - HAProxy** — roundrobin جلوی دو RGW
