---
title: nginx+geoip2+docker实现禁止某个地区或国家访问
categories: Kubernetes
sage: false
date: 2019-10-28 12:10:54
tags: nginx
---

<amp-auto-ads type="adsense" data-ad-client="ca-pub-5216394795966395"></amp-auto-ads>

## nginx 部署网站禁止访问方式

安装docker [参考](https://www.flftuu.com/2019/05/10/debian9-8-docker-ce/)

安装nginx-geoip2 服务

```bash
docker run -d  --name nginx flftuu/nginx-geoip2:1.15.12

```

<!-- more -->

## geoip2 配置禁止访问

```bash
cat nginx/default.conf

server {
    listen      443           ssl http2;
    listen [::]:443           ssl http2;

    add_header                Strict-Transport-Security "max-age=31536000" always;

    ssl_session_cache         shared:SSL:20m;
    ssl_session_timeout       10m;

    ssl_protocols             TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers               "ECDH+AESGCM:ECDH+AES256:ECDH+AES128:!ADH:!AECDH:!MD5;";

    ssl_stapling              on;
    ssl_stapling_verify       on;
    resolver                  8.8.8.8 8.8.4.4;

    root /var/www/html;
    index index.php;

    if ( $geoip2_data_country_code = CN ) {
        return 403;
    }

    if ( $geoip2_data_city_name = Zhengzhou ) {
        return 403;
    }


```

>1. geoip2_data_country_code 设置国家代码
>2. geoip2_data_city_name 设置城市代码

geoip2更多配置[参考](https://dev.maxmind.com/geoip/geoip2/geolite2/)