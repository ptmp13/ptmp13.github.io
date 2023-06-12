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

Cкачаваем src rpm для нужного нам дистрибутива, в нашем случае:

[https://nginx.org/packages/centos/7/SRPMS/](https://nginx.org/packages/centos/7/SRPMS/)

Скачаваем патчи/модули, в нашем случае:

[https://github.com/chobits/ngx_http_proxy_connect_module](https://github.com/chobits/ngx_http_proxy_connect_module)

### Extract files from rpm + download source

Для сборки чего либо по факту достаточно 2х файлов:
_spec_ + _source_

_spec_ - описание как мы будем собирать  
_source_ - исходники из которых мы будем собирать

Но в нашем случае нам надо извлечить все из rpm и закинуть в 
[репу](https://build.opensuse.org/package/show/home:ptmp13/nginx).
Проще всего это сделать так

```bash
rpm2cpio nginx.src.rpm|cpio -idcmv
```

Этой командой мы "распакуем" rpm и получим все необходимые файлы.

### Upload files to buildService

Загружаем файлы которые мы извлекли предыдущей коммандой в репозиторий.
В итоге там должно быть что-то вида:

![img](/img/openSuse-build/SCR-20230612-mdoo.png)

### Build

Сборка запустится автоматом. Можно запустить руками.

Так же вы можете выбрать список дитрибутивов для которых хотите собирать. Делается это на странице:


https://build.opensuse.org/projects/home:<USERNAME>/distributions/new
(Вместо USERNAME - подставить имя вашего пользователя)


# Add modules

Эта часть для тех кто хочет докинуть какой то модуль

### Upload additional module to OpenSuse build Service

Скачивате модуль в нашем случае:
[https://github.com/chobits/ngx_http_proxy_connect_module](https://github.com/chobits/ngx_http_proxy_connect_module)

Проще всего склонировать репу:
```bash
git clone https://github.com/chobits/ngx_http_proxy_connect_module
```
Запаковать скаченную репу в архив:
```bash
tar -czvf ngx_http_proxy_connect_module.tar.gz ngx_http_proxy_connect_module
```

upload to build service

![img](/img/openSuse-build/SCR-20230612-mhdt.png)

так же я залил туда файл которыйц нужен для моей версии nginx те
__proxy_connect.patch__ (proxy_connect_rewrite_102101.patch в скаченной репе)

### Modify Spec for add module

в нашем случае nginx.spec

Для того что бы собирать доп модули надо поправить spec file.
(тут я не претендую на правильность... я в целом изрядно задолбался перебирать врианты и вообщем то остановился на том который просто сработал... потом я разберусь как правильно но не сегодня...)

Собственно нам надо указать в configure что у нас есть доп модуль.. 
+
Распаковать залитый на на преведущем шаге архив ngx_http_proxy_connect_module.tar.gz 

Для этого правим spec следующим образом:
```bash
%define BASE_CONFIGURE_ARGS $(echo "--prefix=%{_sysconfdir}/nginx --sbin-path=%{_sbindir}/nginx --modules-path=%{_libdir}/nginx/modules --conf-path=%{_sysconfdir}/nginx/nginx.conf --error-log-path=%{_localstatedir}/log/nginx/error.log --http-log-path=%{_localstatedir}/log/nginx/access.log --pid-path=%{_localstatedir}/run/nginx.pid --lock-path=%{_localstatedir}/run/nginx.lock --http-client-body-temp-path=%{_localstatedir}/cache/nginx/client_temp --http-proxy-temp-path=%{_localstatedir}/cache/nginx/proxy_temp --http-fastcgi-temp-path=%{_localstatedir}/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=%{_localstatedir}/cache/nginx/uwsgi_temp --http-scgi-temp-path=%{_localstatedir}/cache/nginx/scgi_temp --user=%{nginx_user} --group=%{nginx_group} --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --add-module=./ngx_http_proxy_connect_module")
..........
%build
echo "=====${PWD}====="
tar -xvf /home/abuild/rpmbuild/SOURCES/ngx_http_proxy_connect_module.tar -C .
echo "---------"
ls -la
```

Первая строка длинная но по факту в
__BASE_CONFIGURE_ARGS__ мы просто добавляем модули - __--add-module=./ngx_http_proxy_connect_module__

### Build

Перезаливаем спеку и оно само начнет билдится.

### List modules

```bash
nginx -V
--
strings /usr/sbin/nginx|grep _module|grep -v configure| sort
```

## My links

[OpenSuse build Service PTMP13](https://build.opensuse.org/package/show/home:ptmp13/nginx)  

[https://github.com/ptmp13/openSuseBuildService_nginx](https://github.com/ptmp13/openSuseBuildService_nginx)

## Lifehack

Нужно собирать в идеале для CentOS7 тк для RHEL7 ни фига пакетов может не быть...
