<div dir="rtl" lang="fa">

# HAProxy جلوی چند RGW

آخر جلسهٔ نهم روی همان `ceph-1` (hoodadcloud، شبکهٔ `185.55.227.x`) فایل `/etc/haproxy/haproxy.cfg` را برای load-balance دو `radosgw` ویرایش کردند. این لاب جدا از Glance است؛ مفهوم RGW در `10 - RGW` است.

سرویس در این جلسه **start نشد**. کانفیگ را نگه دارید، نتیجه را با خطاهای journalctl بخوانید.

## کانفیگ لاب

فایل ۴۴ خط / ۱۵۰۲ بایت بود. بخش HTTP که روی صفحه ماند:

```text
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

frontend http_web *:8080
    mode http
    default_backend rgw

backend rgw
    balance roundrobin
    mode http
    server rgw1 185.55.227.42:8080 check
    server rgw2 185.55.227.141:8080 check
```

- کلاینت به **پورت ۸۰۸۰ روی همین نود** وصل می‌شود.
- backend دو RGW با **roundrobin** و health check.
- آدرس‌ها با MONهای همین `ceph.conf` هم‌خانواده‌اند: `.16` / `.42` / `.141`.

لینک چت برای auth RGW:

```text
https://docs.ceph.com/en/latest/radosgw/configuration/auth-config-ref/
```

## تلاش برای start

```bash
service haproxy start
```

```text
Job for haproxy.service failed because the control process exited with error code.
See "systemctl status haproxy.service" and "journalctl -xe" for details.
```

```bash
journalctl -xeu
```

بدون `-u` و اسم واحد قبول نمی‌شود. بعد:

```bash
journalctl -xe
```

نمونهٔ خطا:

```text
haproxy.service: Failed to start HAProxy Load Balancer.
Start request repeated too quickly.
The unit haproxy.service has entered the 'failed' state with result 'exit-code'.
```

دانشجو در چت پرسید مطمئنی HAProxy روی این نود اجرا می‌شود؟ و گفت `journalctl -xe` را ببین کجا error داده. در همین لاگ SSH با پسورد اشتباه از `49.88.112.113` و scrape پرومتهوس روی `185.55.227.16` هم دیده شد؛ آن‌ها علت fail HAProxy نیستند.

`iptables -L` زنجیره‌های DOCKER را نشان داد (cephadm روی همین میزبان container دارد). اگر پورت ۸۰۸۰ را خود RGW یا چیز دیگری bind کرده باشد، HAProxy بالا نمی‌آید.

> **هشدار:** لاب این جلسه با HAProxy سالم تمام نشد. قبل از `systemctl start haproxy` با `ss -lntp | grep 8080` ببینید پورت آزاد است، بعد `haproxy -c -f /etc/haproxy/haproxy.cfg` را برای syntax بزنید.
</div>
