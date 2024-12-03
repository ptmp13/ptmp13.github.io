---
layout:     post
title:      "Apache Httpd OpenID + Keycloak..."
subtitle:   " \"Tratatata... \""
date:       2024-11-20 17:49:00
author:     "ptmp13"
catalog: true
tags:
    - Httpd
    - Apache
    - OpenID
    - Keycloak
---

## Что нам понадобится.

1. Keycloak - Короче конкретно с keycloak заморачиваться мы не будем типа dev mode и вперед.  
2. Httpd с модулем openid а именно [mod_auth_openidc.so](https://github.com/OpenIDC/mod_auth_openidc)
3. Ну бразер очевидно... типа firefox

## KeyCloak

Запускать его мы будем в docker/podman т.е командой:
```bash
podman run -p 8080:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:24.0.4 start-dev
```

В итоге оно напишет чет типа

```txt
 WARN  [org.keycloak.quarkus.runtime.KeycloakMain] (main) Running the server in development mode. DO NOT use this configuration in production.
```

Далее че надо забацать там какой рилм там пользака чисто для теста. Для этого заходим в админку __http://keycloakhost:8080__  

_login: admin_  
_pass: admin_  

Создаем тестовый realm, так и назовем TestRealm  
![img](/img/in-post/http-keycloak/CreateRealm.png)

Далее саданем туда клиента ну типа что бы сгенерить ему всякие штуки

Тут вбиваем имя клиента и саданем галочку всегда показывать в UI (хрен знает что она значит... но навреное было бы не плохо если бы он всегда был виден)  

![img](/img/in-post/http-keycloak/CreateClient-1.png)

_Next_

Тут включаем Client authentication потому что httpd должен будет как то общаться с нашим keycloak и надо будет сгенерить ему данные для аутентификации.  

![img](/img/in-post/http-keycloak/CreateClient-2.png)

_Next_

Тут указывем URL с которого мы будем "заходить" например http://lol.corp.local  

![img](/img/in-post/http-keycloak/CreateClient-3.png)

_Save_

Далее надо создать какого нить юзера для теста  

![img](/img/in-post/http-keycloak/CreateUser-1.png)

Меняем ему пароль и убираем "временный"  

![img](/img/in-post/http-keycloak/CreateUser-2.png)


## Httpd

Самый простой вариант закидываем куда нить в __conf.d__ конфиг вида

```apacheconf
    OIDCProviderMetadataURL http://trolol.corp.local:8080/realms/TestRealm/.well-known/openid-configuration
    OIDCClientID httpd-test
    OIDCClientSecret tC2YAYkxk1VHI6D0EAUrYYYcagvZmosT
    OIDCProviderTokenEndpointAuth client_secret_basic
    OIDCRedirectURI /restricted
    OIDCCryptoPassphrase OLOLOL
    OIDCRemoteUserClaim preferred_username
    ### Optional if OIDCProviderMetadataURL https
    OIDCCABundlePath /usr/local/apache2/ssl/ca-root2.pem

    DocumentRoot /var/www/html/

    <Location /restricted>
        AuthType openid-connect
        Require claim read_restricted:true
    </Location>
```

Соответственно что бы "прикрыть" location

```apacheconf
    <Location "/azaz">
        AuthType openid-connect
        Require valid-user
    </Location>
```

Чет надоело писать потом допишу еще че нить...