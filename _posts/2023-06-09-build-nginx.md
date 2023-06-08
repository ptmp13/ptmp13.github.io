---
layout:     post
title:      "Build nginx with openSuse Build Service..."
subtitle:   " \"... to later...\""
date:       2023-06-08 16:51:00
author:     "ptmp13"
catalog: true
tags:
    - Linux
    - rpm
    - build
    - nginx
---

# Lets go

Я не знаю как билдить что-то универсальное поэтому будем отталкиваться от имеющейся проблемы.
А именно нам нужен nginx для RHEL7

### Загрузка необходимых вещей

Перво наперво надо скачать src rpm для нужного нам дистрибутива в случае nginx это можно сделать например сдесь

[https://nginx.org/packages/centos/7/SRPMS/](https://nginx.org/packages/centos/7/SRPMS/)

Потенциально можно скачать какие то патчи модули или что там нам еще надо в нашем случае например:

[https://github.com/chobits/ngx_http_proxy_connect_module](https://github.com/chobits/ngx_http_proxy_connect_module)

### Extract files from rpm + download source

Для сборки чего либо по факту достаточно 2х файлов (скажем так минимальный набор)
_spec_ + _source_

_spec_ - описание как мы будем собирать
_source_ - исходники из которых мы будем собирать

Но в нашем случае нам надо извлечить все из rpm и закинуть в репу
в openSuse Build Server 
Проще всего это сделать так

```bash
rpm2cpio nginx.src.rpm|cpio -idcmv
```

### Upload files to buildService

Собственно че надо зафигачить туда все что нам надо
Тут перечислять не буду можно посмотреть  моей репе
[github](github)

### Build

Сборка запустится автоматом и тут проблем быть не должно все соберется ок.

# Add modules

Эта часть для тех кто хочет докинуть какой то модуль

### Upload additional module to OpenSuse build Service

Что бы докинуть модули нужны сами модули на примере
[https://github.com/chobits/ngx_http_proxy_connect_module](https://github.com/chobits/ngx_http_proxy_connect_module)

мы можем качнуть репу через
```bash
git clone 
```
Дальше нам надо закинуть что в build Service ну и поправить спеку...

Файлы что закинуть первое собственно запаковать скаченную репу
```bash
tar -czvf ngx_http_proxy_connect_module.tar ngx_http_proxy_connect_module
```

Дальше upload to build service
так же я залил туда файл которыйц нужен для моей версии nginx те
__proxy_connect.patch__ (proxy_connect_rewrite_102101.patch в скаченной репе)

### Modify Spec for add module

nginx.spec - в нашем случае...

Тут значит нам надо чет поправить в спеке
(на самом деле я не претендую на правильность какую то ибо ну его... я в целом изрядно задолбался перебирать врианты и вообщем то остановился на том который просто сработал... потом я точно разберусь как правильно но не сегодня...)

Собственно так же нам надо указать в configure что у нас есть доп модуль но тут не понятно где он нахрен лежит поэтому.. мы его наглым образом растарим куда надо

```txt
%define BASE_CONFIGURE_ARGS $(echo "--prefix=%{_sysconfdir}/nginx --sbin-path=%{_sbindir}/nginx --modules-path=%{_libdir}/nginx/modules --conf-path=%{_sysconfdir}/nginx/nginx.conf --error-log-path=%{_localstatedir}/log/nginx/error.log --http-log-path=%{_localstatedir}/log/nginx/access.log --pid-path=%{_localstatedir}/run/nginx.pid --lock-path=%{_localstatedir}/run/nginx.lock --http-client-body-temp-path=%{_localstatedir}/cache/nginx/client_temp --http-proxy-temp-path=%{_localstatedir}/cache/nginx/proxy_temp --http-fastcgi-temp-path=%{_localstatedir}/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=%{_localstatedir}/cache/nginx/uwsgi_temp --http-scgi-temp-path=%{_localstatedir}/cache/nginx/scgi_temp --user=%{nginx_user} --group=%{nginx_group} --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --add-dynamic-module=./ngx_http_proxy_connect_module --add-dynamic-module=./ngx_http_proxy_connect_module --add-module=./nginx-module-vts --add-module=./ngx_cache_purge --add-module=./spnego-http-auth-nginx-module")
..........
%build
echo "=====${PWD}====="
tar -xvf /home/abuild/rpmbuild/SOURCES/ngx_http_proxy_connect_module.tar -C .
echo "---------"
ls -la
```

Ну вообщем то и все

### List modules

```bash
nginx -V
--
strings /usr/sbin/nginx|grep _module|grep -v configure| sort
```

## My links

[OpenSuse build Service PTMP13](https://build.opensuse.org/package/show/home:ptmp13/nginx)  

[https://github.com/ptmp13/openSuseBuildService_nginx](https://github.com/ptmp13/openSuseBuildService_nginx)
