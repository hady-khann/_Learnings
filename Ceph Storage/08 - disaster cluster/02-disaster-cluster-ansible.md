<div dir="rtl" lang="fa">

# ساخت کلاستر Disaster با ceph-ansible

کلاستر Passive روی نودهای `ceph-node5` تا `ceph-node7` با همان `ceph-ansible` کلاستر اصلی ساخته می‌شود؛ فقط Inventory و نام Cluster فرق دارد.

## ۱. Inventory کلاستر Active (مرجع)

روی `ceph-node1` فایل Ansible hosts چیزی شبیه این است:

```text
[mons]
ceph-node1
ceph-node2
ceph-node3

[osds]
ceph-node1
ceph-node2
ceph-node3

[grafana-server]
ceph-node1

[clients]
client-node1
```

## ۲. Inventory کلاستر Disaster

روی `ceph-node5` همین ساختار برای سایت Passive:

```text
[mons]
ceph-node5
ceph-node6
ceph-node7

[osds]
ceph-node5
ceph-node6
ceph-node7

[grafana-server]
ceph-node5
```

Inventory یک فایل است، نه دایرکتوری:

```bash
cat /etc/ansible/hosts
```

```bash
cd /etc/ansible/hosts
```

خطای زیر یعنی مسیر را به‌اشتباه به‌عنوان پوشه باز کرده‌اید:

```text
-bash: cd: /etc/ansible/hosts: Not a directory
```

## ۳. نام Cluster = backup

در `group_vars/all.yml` کلاستر disaster، مقدار `cluster` را از پیش‌فرض `ceph` به `backup` تغییر دهید:

```yaml
cluster: backup
```

با این کار فایل‌های تنظیمات در `/etc/ceph` این شکل می‌شوند:

```text
backup.conf
backup.client.admin.keyring
```

به‌جای `ceph.conf` / `ceph.client.admin.keyring`.

## ۴. دیسک OSD

در `group_vars/osds.yml` دستگاه‌های Block مشخص می‌شوند؛ در لاب:

```yaml
devices:
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
# - /dev/sde
```

## ۵. وضعیت کلاستر Disaster

بعد از نصب، چون نام Cluster برابر `backup` است:

```bash
ceph -s --cluster backup
```

یا:

```bash
export CEPH_ARGS="--cluster backup"
ceph -s
```

خروجی مورد انتظار چیزی شبیه این است (نودها و FSID لاب):

```text
cluster:
  id:     f5efec14-2866-48f2-bfc7-01ee9ef11358
  health: HEALTH_WARN

services:
  mon: 3 daemons, quorum ceph-node5,ceph-node6,ceph-node7
  osd: 9 osds: 9 up, 9 in
```

`HEALTH_WARN` در ویدیو به‌خاطر مواردی مثل `insecure global_id reclaim` و clock skew روی `mon.ceph-node6` / `mon.ceph-node7` بود؛ برای ادامهٔ کار mirroring الزاماً مانع نیست.

## ۶. دسترسی SSH بین دو سایت

از نود disaster به نود primary کلید بفرستید:

```bash
nano /etc/hosts
ssh-copy-id root@ceph-node1
```

در لاب آدرس `ceph-node1` برابر `192.168.1.15` بود.

## نکتهٔ CEPH_ARGS در bashrc

اگر نام Cluster همیشه `backup` است، می‌توان `export CEPH_ARGS="--cluster backup"` را در `~/.bashrc` گذاشت تا هر بار `ceph -s` بدون فلگ کار کند.
</div>
