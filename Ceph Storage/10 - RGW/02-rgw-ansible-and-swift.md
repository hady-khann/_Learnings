<div dir="rtl" lang="fa">

# نصب RGW با ceph-ansible و تست Swift

نود `rgw-node1` (`192.168.1.11`) را به Inventory اضافه کنید، Playbook را دوباره اجرا کنید، کاربر Object بسازید، و از `client-node1` با CLI `swift` یک bucket بسازید.

## ۱. Hostname نود RGW

روی خود `rgw-node1` در `/etc/hosts`:

```text
127.0.0.1   localhost  rgw-node1
127.0.1.1   rgw-node1
192.168.1.11 rgw-node1
```

روی `ceph-node1` هم نام را در `/etc/hosts` بگذارید تا Ansible به آن SSH بزند.

## ۲. Inventory و all.yml

گروه Ansible برای Gateway معمولاً `[rgws]` است. در خروجی Playbook نودهای زیر دیده شد:

```text
[ceph-node1] [ceph-node2] [ceph-node3] [rgw-node1] [client-node1]
```

در `~/ceph-ansible/group_vars/all.yml` پورت Civetweb را ۸۰۸۰ بگذارید (اگر کامنت است، از حالت comment درآورید):

```yaml
radosgw_civetweb_port: 8080
```

سپس از دایرکتوری `ceph-ansible`:

```bash
cd ~/ceph-ansible
ansible-playbook site.yml
```

هشدارهای `Could not match supplied host pattern` برای گروه‌هایی مثل `nfss`، `rbdmirrors`، `iscsigws` اگر در Inventory خالی باشند طبیعی است.

بعد از موفقیت:

```bash
ceph -s
ceph osd pool ls
```

باید `rgw: 1 daemon active (rgw-node1.rgw0)` و Poolهای `.rgw.root`، `default.rgw.log`، `default.rgw.control`، `default.rgw.meta` را ببینید.

روی `rgw-node1` در `/etc/ceph` در ابتدا فقط `ceph.conf` و `rbdmap` بود؛ keyring daemon اینجاست:

```text
/var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring
```

```text
[client.rgw.rgw-node1.rgw0]
        key = AQCQtHNjx287DRAAq0+q2ub4EPZ4F+UIieeJ8w==
```

## ۳. ساخت کاربر Object — ترتیب مهم است

اول **user**، بعد **subuser**. در لاب اول `subuser create` زدند و این خطا آمد:

```text
could not create subuser: unable to parse request, user info was not populated
```

یعنی `uid=anisa-rgw` هنوز وجود ندارد.

روی `rgw-node1`:

```bash
radosgw-admin user create \
  --uid=anisa-rgw \
  --display-name="Iran Linux House" \
  --email=info@anisa.co.ir \
  --access=full \
  -k /var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring \
  --name client.rgw.rgw-node1.rgw0
```

خروجی JSON شامل `keys` (S3 access/secret) و سهمیه‌های `bucket_quota` / `user_quota` با `"enabled": false` است.

سپس subuser برای Swift:

```bash
radosgw-admin subuser create \
  --uid=anisa-rgw \
  --subuser=anisa-rgw:swift \
  --access=full \
  -k /var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring \
  --name client.rgw.rgw-node1.rgw0
```

اطلاعات کاربر:

```bash
radosgw-admin user info --uid=anisa-rgw \
  -k /var/lib/ceph/radosgw/ceph-rgw.rgw-node1.rgw0/keyring \
  --name client.rgw.rgw-node1.rgw0
```

از JSON، `swift_keys` را بردارید. در لاب secret Swift این بود:

```text
rwWMVDpLqVqwDvMS00frFCT9c7I6hAhF1Ito3hIG
```

> **توجه:** `-k` باید به **فایل** `keyring` اشاره کند، نه به پوشهٔ `.../ceph-rgw.rgw-node1.rgw0`. اگر پوشه بدهید:

```text
auth: error reading file: ... (21) Is a directory
```

## ۴. تست Swift از کلاینت

روی `client-node1` (ابزار `swift` باید نصب باشد):

لیست containerها (اول خالی است؛ ممکن است usage کامل چاپ شود):

```bash
swift -A http://192.168.1.11:8080/auth/1.0 \
  -U anisa-rgw:swift \
  -K rwWMVDpLqVqwDvMS00frFCT9c7I6hAhF1Ito3hIG \
  list
```

ساخت bucket/container:

```bash
swift -A http://192.168.1.11:8080/auth/1.0 \
  -U anisa-rgw:swift \
  -K rwWMVDpLqVqwDvMS00frFCT9c7I6hAhF1Ito3hIG \
  post anisa-bucket
```

دوباره `list`:

```text
anisa-bucket
```

`-A` آدرس Auth، `-U` همان subuser، `-K` کلید Swift است.

S3 با `s3cmd` در چت کلاس مطرح شد ولی در این جلسه CLI Swift اجرا شد. SDKهایی مثل AWS PHP هم به همان endpoint روی پورت ۸۰۸۰ وصل می‌شوند.

## ۵. فضای مصرف‌شده بعد از RGW

`ceph df` بعد از بالا آمدن Gateway Poolهای جدید را نشان می‌دهد؛ مثلاً `default.rgw.log` چند مگابایت log دارد حتی قبل از بار واقعی. Filesystem CephFS هم در همان خروجی هست (`cephfs_data` / `cephfs_metadata`) — RGW آن‌ها را جایگزین نمی‌کند.
</div>
