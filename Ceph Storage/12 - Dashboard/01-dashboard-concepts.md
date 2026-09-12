# Ceph Dashboard و Grafana

نیمهٔ دوم جلسهٔ هشتم Dashboard را روی همان کلاستر لاب (`ceph-node1` = `192.168.1.15`) بالا می‌آورد. Grafana/Prometheus از قبل به‌صورت Container روی نود بودند؛ مشکل اصلی این بود که پورت ۸۴۴۳ جواب نمی‌داد تا پکیج `ceph-mgr-dashboard` و نسخهٔ Pacific درست شود.

Hardware این جلسه در پوشهٔ `11 - Hardware` است. ساخت کاربر CephX (`client.shobeyr`) فقط مرور `05 - Cephx` بود؛ اینجا تکرار نمی‌شود.

## وضعیت کلاستر قبل از کار

روی `ceph-node1`:

```text
cluster:
  id:     9f0fca15-8d21-4bb2-9512-57867f3dabb2
  health: HEALTH_WARN
          1 pool(s) do not have an application enabled
          2 pool(s) have non-power-of-two pg_num

  services:
    mon: 3 daemons, quorum ceph-node1,ceph-node2,ceph-node3
    mgr: ceph-node1(active), standbys: ceph-node2, ceph-node3
    mds: cephfs:1 {0=ceph-node2=up:active}
    osd: 9 osds: 9 up, 9 in

  data:
    pools:   6 pools, 197 pgs
    objects: 81 objects, 17 MiB
    usage:   9.2 GiB used, 171 GiB / 180 GiB avail
    pgs:     197 active+clean
```

شش Pool یعنی RGW این snapshot روی کلاستر نبود (یا پاک شده بود). `HEALTH_WARN` به‌خاطر application و `pg_num` است، نه به‌خاطر خود Dashboard.

## Containerهای مانیتورینگ

`docker ps` روی `ceph-node1`:

```text
grafana/grafana:5.4.3          grafana-server
prom/prometheus:v2.7.2         prometheus
prom/alertmanager:v0.16.2      alertmanager
prom/node-exporter:v0.17.0     node-exporter
```

Grafana از قبل روی پورت ۳۰۰۰ باز بود:

```text
http://192.168.1.15:3000
```

داشبورد `ceph-dashboard / Host Overview` در لاب این ارقام را نشان داد: ۳ OSD Host، CPU حدود ۵.۶٪، RAM حدود ۳۸٪، IOPS حدود ۱۴۲.

> **هشدار:** خودِ Ceph Dashboard (ماژول MGR روی ۸۴۴۳) با Grafana یکی نیست. Grafana متریک را می‌کشد؛ Dashboard مدیریت کلاستر است و باید API Grafana/Prometheus را جدا به آن بدهید.
