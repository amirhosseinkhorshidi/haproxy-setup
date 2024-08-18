# راهنمای نصب و پیکربندی HAProxy برای Load Balancing

این راهنما به شما کمک می‌کند تا [HAProxy](https://docs.haproxy.org/) را به عنوان یک لود بالانسر برای توزیع ترافیک بین چندین سرور نصب و پیکربندی کنید. HAProxy یک نرم‌افزار open-source و بسیار پرکاربرد برای load balancing و proxying است که می‌تواند ترافیک شبکه را به صورت موثر بین چندین سرور توزیع کند. در این راهنما، پیکربندی یک load balancer برای توزیع ترافیک HTTPS و HTTP بین چند سرور نمونه را آموزش خواهیم داد.

## ۱. نصب HAProxy

برای نصب HAProxy بر روی سرور Ubuntu، از دستور زیر استفاده کنید:

```bash
sudo apt update  
sudo apt install haproxy -y
```

پس از نصب، فایل پیکربندی HAProxy در مسیر `/etc/haproxy/haproxy.cfg` قرار دارد که باید آن را ویرایش کنیم.

## ۲. پیکربندی HAProxy

### بخش Global

در بخش `global`، تنظیمات کلی مربوط به HAProxy مشخص می‌شود. این تنظیمات بر روی تمام پروسس‌های HAProxy اعمال می‌شود:

```haproxy
global  
    log /dev/log    local0  
    log /dev/log    local1 notice  
    chroot /var/lib/haproxy
```

### بخش Defaults

در بخش `defaults`، تنظیمات پیش‌فرض برای frontendها و backendها مشخص می‌شود. این تنظیمات شامل تنظیمات مربوط به لاگ‌ها، تایم‌اوت‌ها، و گزینه‌های مربوط به TCP و HTTP است:

```haproxy
defaults  
    log global  
    retry-on all-retryable-errors  
    timeout connect 5s  
    timeout client 50s  
    timeout client-fin 50s  
    timeout server 50s  
    timeout tunnel 1h  
    default-server init-addr none  
    default-server inter 15s fastinter 2s downinter 5s rise 3 fall 3  
    option tcpka  
    option tcp-smart-connect  
    option tcp-smart-accept  
    option dontlognull  
    option http-keep-alive  
    errorfile 403 /etc/haproxy/errors/403.http
```

در این بخش، ترافیک HTTPS که به پورت 443 سرور وارد می‌شود، بین دو سرور backend تقسیم می‌شود. این پیکربندی به HAProxy اجازه می‌دهد تا ترافیک SSL/TLS را به درستی مدیریت کند:

```haproxy
frontend https_front  
    bind :::443 v4v6  
    mode tcp

    # Rules  
    tcp-request inspect-delay 5s  
    tcp-request content accept if { req_ssl_hello_type 1 }

    default_backend proxy_servers

backend proxy_servers  
    mode tcp  
    server srv1 192.168.1.1:443 check  
```

- بررسی **frontend `https_front`**: این بخش درخواست‌های ورودی به پورت 443 را دریافت کرده و به backend مربوطه هدایت می‌کند.
- بررسی **backend `proxy_servers`**: این بخش شامل لیست سرورهایی است که درخواست‌ها بین آنها توزیع می‌شود. در اینجا یک سرور `srv1` با آدرس IPv4 تعریف شده‌اند. برای استفاده از آدرس‌های IPv6 باید درون کروشه `[]` قرار گیرد.

### استفاده از Load Balancing با الگوریتم Round-Robin

در این بخش، ترافیک بین سه سرور (دو IPv4 و یک IPv6) به صورت round-robin توزیع می‌شود. الگوریتم Round-Robin یکی از رایج‌ترین الگوریتم‌های load balancing است که درخواست‌ها را به صورت چرخشی بین سرورها توزیع می‌کند:

```haproxy
frontend https_front_rr  
    bind :::443 v4v6  
    mode tcp

    default_backend rr_servers

backend rr_servers  
    mode tcp  
    balance roundrobin  
    server srv1 192.168.1.1:443 check  
    server srv2 192.168.1.2:443 check  
    server srv3 [2a03:b0c0:3:d0::1890:2002]:443 check
```

- بررسی **frontend `https_front_rr`**: این بخش درخواست‌های ورودی به پورت 443 را دریافت کرده و به backend مربوطه هدایت می‌کند.
- بررسی **backend `rr_servers`**: این بخش از الگوریتم `roundrobin` برای توزیع درخواست‌ها بین سه سرور استفاده می‌کند:
  - سرور اول `srv1` با آدرس IPv4
  - سرور دوم `srv2` با آدرس IPv4
  - سرور سوم `srv3` با آدرس IPv6 (درون کروشه `[]`)

### مدیریت ترافیک HTTP

در این بخش، ترافیک HTTP که به پورت 80 وارد می‌شود، به یک backend خاص هدایت شده و سپس درخواست‌ها بر اساس قوانین تعریف شده مدیریت می‌شود:

```haproxy
frontend http_front  
    bind :::80 v4v6  
    mode http

    # frontend Rules  
    default_backend blackhole_http

backend blackhole_http  
    mode http  
    errorfile 403 /etc/haproxy/errors/403.http  
    http-request deny deny_status 403
```

- بررسی **frontend `http_front`**: این بخش درخواست‌های ورودی به پورت 80 را دریافت کرده و آنها را به backend `blackhole_http` هدایت می‌کند.
- بررسی **backend `blackhole_http`**: این بخش به گونه‌ای پیکربندی شده است که تمام درخواست‌های HTTP را با کد وضعیت 403 رد می‌کند. این می‌تواند برای مسدود کردن ترافیک ناخواسته یا غیرمجاز استفاده شود.

## ۳. مسیر دهی (ACLs) در HAProxy

در این بخش، به نحوه مسیر‌دهی درخواست‌ها بر اساس مسیر (Path)، نام سرور (SNI) و پورت پرداخته خواهد شد.

### مسیر‌دهی بر اساس Path

در این مثال، درخواست‌های به `/api` به یک backend خاص هدایت می‌شوند:

```haproxy
frontend http_front  
    bind :::80 v4v6  
    mode http

    acl is_api_path path_beg /api  
    use_backend api_backend if is_api_path

backend api_backend  
    mode http  
    server srv1 192.168.1.1:80 check  
    server srv2 192.168.1.2:80 check
```

- بررسی **acl `is_api_path`**: این ACL بررسی می‌کند که آیا مسیر درخواست با `/api` شروع می‌شود یا خیر.
- بررسی **use_backend `api_backend`**: اگر شرط ACL برقرار باشد، درخواست به backend مربوطه هدایت می‌شود.

### مسیر‌دهی بر اساس SNI (ریشه دامین)

در این مثال، درخواست‌های HTTPS بر اساس نام دامنه به backend‌های مختلف هدایت می‌شوند:

```haproxy
frontend https_front  
    bind :::443 v4v6  
    mode tcp

    acl is_example_com req.ssl_sni -i example.com  
    use_backend example_com_backend if is_example_com

backend example_com_backend  
    mode tcp  
    server srv1 192.168.1.1:443 check  
    server srv2 192.168.1.2:443 check
```

- بررسی **acl `is_example_com`**: این ACL بررسی می‌کند که آیا SNI در درخواست برابر با `example.com` است یا خیر.
- بررسی **use_backend `example_com_backend`**: اگر شرط ACL برقرار باشد، درخواست به backend مربوطه هدایت می‌شود.

### مسیر‌دهی بر اساس SNI (ساب‌دامین)

در این مثال، درخواست‌هایی که به ساب‌دامین خاصی ارسال می‌شوند، به backend‌های مختلف هدایت می‌شوند:

```haproxy
frontend https_front  
    bind :::443 v4v6  
    mode tcp

    acl is_sub_example_com req.ssl_sni -i .*\.example\.com$
    use_backend sub_example_com_backend if is_sub_example_com

backend sub_example_com_backend  
    mode tcp  
    server srv1 192.168.1.3:443 check  
    server srv2 [2a03:b0c0:3:d0::1890:2003]:443 check
```

- بررسی **acl `is_sub_example_com`**: این ACL بررسی می‌کند که آیا SNI در درخواست برابر با `sub.example.com` است یا خیر.
- بررسی **use_backend `sub_example_com_backend`**: اگر شرط ACL برقرار باشد، درخواست به backend مربوطه هدایت می‌شود.

### مسیر‌دهی بر اساس پورت

در این مثال، درخواست‌هایی که به پورت 8080 ارسال می‌شوند، به یک backend خاص هدایت می‌شوند:

```haproxy
frontend custom_port_front  
    bind :::8080 v4v6  
    mode tcp

    acl is_custom_port dst_port 8080  
    use_backend custom_port_backend if is_custom_port

backend custom_port_backend  
    mode tcp  
    server srv1 192.168.1.4:8080 check  
    server srv2 192.168.1.5:8080 check
```

- بررسی **acl `is_custom_port`**: این ACL بررسی می‌کند که آیا درخواست به پورت 8080 ارسال شده است یا خیر.
- بررسی **use_backend `custom_port_backend`**: اگر شرط ACL برقرار باشد، درخواست به backend مربوطه هدایت می‌شود.

## ۴. بررسی پیکربندی و راه‌اندازی مجدد HAProxy

پس از اعمال تغییرات در فایل پیکربندی، برای بررسی صحت پیکربندی و راه‌اندازی مجدد HAProxy از دستورات زیر استفاده کنید:

```bash
sudo haproxy -f /etc/haproxy/haproxy.cfg -c  
sudo systemctl restart haproxy
```

دستور اول پیکربندی را برای خطاها بررسی می‌کند و دستور دوم سرویس HAProxy را با پیکربندی جدید راه‌اندازی مجدد می‌کند.

## ۵. نظارت بر HAProxy

برای نظارت بر عملکرد HAProxy و بررسی لاگ‌ها، می‌توانید از دستورات زیر استفاده کنید:

```bash
sudo tail -f /var/log/haproxy.log
```

این دستور لاگ‌های مربوط به HAProxy را نمایش می‌دهد و به شما امکان می‌دهد تا مشکلات یا خطاهای احتمالی را شناسایی کنید.
