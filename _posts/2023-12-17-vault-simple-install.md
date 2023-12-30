---
layout:     post
title:      "Vault ha (raft storage)..."
subtitle:   " \"...\""
date:       2023-12-17 20:49:00
author:     "ptmp13"
catalog: true
tags:
    - Vault
    - Hashicorp
---

> Ok just install it

## Prepare

```bash
ip addr add 127.0.0.2/8 dev lo label lo:0
ip addr add 127.0.0.3/8 dev lo label lo:1

cat /etc/hosts
127.0.0.2 vault2.tmp.local
127.0.0.3 vault3.tmp.local
```

## Prepare Certs

Вообщем то нам полюбому надо выпить рому... точнее сгенерить сертификаты поэтому генерим сертификаты в первую очередь что бы потом не парится с этим

### Gen CA cert

Генерим CA

```bash
openssl req -x509 -sha256 -days 1825 -newkey rsa:2048 -keyout rootCA.key -out rootCA.crt -subj "/CN=vaultca.tmp.local/C=RU/ST=Moscow/L=Moscow/O=ZAO tmp/OU=ITDepartment/"
```

### Create certificate request

Тут надо сделать запрос csr

```bash
openssl req -newkey rsa:2048 -keyout vault.key -out vault.csr -subj "/CN=*.tmp.local/C=RU/ST=Moscow/L=Moscow/O=ZAO tmp/OU=ITDepartment/"
# OR
#openssl genrsa -out vault.key 4096
#openssl req -new -key vault.key -sha256 -out vault.csr -config cert.config
```

### Sign CSR with our CA cert

Вообщем то тут надо бы сгенерить какие нить простенькие сертификаты... но желательно сделать их не на 10 дней потому что иначе получим ошибку типа

> Error checking seal status: Get "https://127.0.0.1:8200/v1/sys/seal-status": x509: certificate has expired or is not yet valid: current time 2023-12-18T15:48:30+03:00 is after 2023-10-27T15:41:51Z


Так же может быть такая ошибка эт потому что нам надо юзать
> Error checking seal status: Get "https://vault2.tmp.local:8200/v1/sys/seal-status": x509: certificate relies on legacy Common Name field, use SANs instead

__-extfile <(printf "subjectAltName=DNS:*.tmp.local")__


```bash
openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in vault.csr -out vault.crt -days 3650 -CAcreateserial -extfile <(printf "subjectAltName=DNS:*.tmp.local")
# Or for fullname hosts
#openssl x509 -req -CA rootCA.crt -CAkey rootCA.key -in vault.csr -out vault.crt -days 3650 -CAcreateserial -extfile <(printf "subjectAltName=DNS.1:vault2.tmp.local,DNS.2:vault3.tmp.local,IP.1:127.0.0.2,IP.2:127.0.0.3")
# OR with file
#openssl x509 -req -sha256 -in vault.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out vault.crt -days 365 -extfile cert.config -extensions 'v3_req'

openssl rsa -in vault.key -out vault_nopass.key
openssl verify -verbose -CAfile rootCA.crt vault.crt
```

Второй командой убираем пароль из ключа. Ибо Vault не умеет пароли, и github предлагают какие то костыли...

[If the key file is encrypted, you will be prompted to enter the passphrase on server startup](https://developer.hashicorp.com/vault/docs/configuration/listener/tcp)

Что бы не было ошибки типа
>Error checking seal status: Get "https://vault2.tmp.local:8200/v1/sys/seal-status": x509: certificate signed by unknown authority

Надо закинуть CA сертификат в наш файлик с сертификатом, для этого делаем следующее + дока говорит - 
[The primary certificate should appear first in the combined](https://developer.hashicorp.com/vault/docs/configuration/listener/tcp#tls_cert_file)
file.

```bash
cat rootCA.crt >> vault.crt
```


## Node ONE

### Simple config for raft

Т.к у нас первый сервер на IP 127.0.0.2 поэтому я для него конфиг назвал server-2.hcl

```hcl
storage "raft" {
  path    = "/media/data/vault2/raft"
  node_id = "oadmintst2"

  retry_join {
    leader_api_addr = "https://vault2.tmp.local:8200"
    leader_ca_cert_file = "/opt/vault/certs/withca/rootCA.crt"
    leader_client_cert_file = "/opt/vault/certs/withca/vault.crt"
    leader_client_key_file = "/opt/vault/certs/withca/vault_nopass.key"
  }
  retry_join {
    leader_api_addr = "https://vault3.tmp.local:8200"
    leader_ca_cert_file = "/opt/vault/certs/withca/rootCA.crt"
    leader_client_cert_file = "/opt/vault/certs/withca/vault.crt"
    leader_client_key_file = "/opt/vault/certs/withca/vault_nopass.key"
  }
}

listener "tcp" {
 address = "vault2.tmp.local:8200"
 cluster_address = "vault2.tmp.local:8201"
 tls_cert_file = "/opt/vault/certs/withca/vault.crt"
 tls_key_file = "/opt/vault/certs/withca/vault_nopass.key"
 tls_client_ca_file = "/opt/vault/certs/withca/rootCA.crt"
 tls_disable = 0
}

ui = true
api_addr = "https://vault2.tmp.local:8200"
cluster_addr = "https://vault2.tmp.local:8201"
cluster_name = "vault-tmp"
log_level = "INFO"
disable_mlock=true
```

> Cluster address must be set when using raft storage

Если видим такую херню то очевидно надо забыли cluster_addr... 

### Run

Запускаем это чудо:
```bash
vault server -config=/etc/vault/conf.d/server-2.hcl
```

Проверяем статус
```bash
export VAULT_ADDR='https://vault2.tmp.local:8200'
export VAULT_CACERT=/opt/vault/certs/withca/rootCA.crt
vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.12.0
Build Date         2022-10-10T18:14:33Z
Storage Type       raft
HA Enabled         true
```

Тут вообщем все намана должно быть...
Но мы видимо что __Initialized: false__ 

Поэтому надо инициализовать наш новый vault
```bash
vault operator init
Unseal Key 1: XPFnXMHpFDqwXK9uCjgtNSw4wlrYkfXRY6QttU1KSsXt
Unseal Key 2: rgTtwxv/Gx84vubOqdvyls9C/IlVWw84E9nplLHe5bhe
Unseal Key 3: 8EyC1p6L701iVs+7c6U4It8P5yk0TWGHV1JJASQHU2z5
Unseal Key 4: /jzdxtVBqPtDQMXls0TcxhoigoA7s4KT/l5VR5vDWNAT
Unseal Key 5: L59iKxsnPHGYc4I83iHeuW0pjujhRN8tm4VJAwXIHFOF

Initial Root Token: hvs.AGXnkD9hJJX2Tf8yzQCp9gNO
```

Дальше надо рассолить это дело
```bash
vault operator unseal # 3раза ибо у нас treshhold 3
```

Тут почему то я подумал на кой хер мне вообще эит 5 ключей и решил переделать на один нах
```bash
vault operator rekey -init -key-threshold=1 -key-shares=1
vault operator rekey
...
#Rekey operation nonce: 939cd1b9-7b23-5bb0-a8e8-2311a1af8937
#Unseal Key (will be hidden):

#Key 1: oQY3TjGmc4OoioVDQG4n2qlgN34MXFdjyumYyezZgIM=

#Operation nonce: 939cd1b9-7b23-5bb0-a8e8-2311a1af8937
```

### Unseal

```bash
vault operator unseal
# insert key
vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.12.0
Build Date              2022-10-10T18:14:33Z
Storage Type            raft
Cluster Name            vault-tmp
Cluster ID              78c82cd4-5ce1-0ca6-4849-6e7c5a5822fb
HA Enabled              true
HA Cluster              https://vault2.tmp.local:8201
HA Mode                 active
Active Since            2023-12-21T06:28:31.593633502Z
Raft Committed Index    78
Raft Applied Index      78
```

### Login + list peer

```bash
vault login
vault operator raft list-peers
Node          Address                  State     Voter
----          -------                  -----     -----
oadmintst2    vault2.tmp.local:8201    leader    true
```

## Node TWO

### Config
Т.к у нас второй сервер на IP 127.0.0.3 поэтому я для него конфиг назвал server-3.hcl

```hcl
storage "raft" {
  path    = "/media/data/vault3/raft"
  node_id = "oadmintst3"

  retry_join {
    leader_api_addr = "https://vault2.tmp.local:8200"
    leader_ca_cert_file = "/opt/vault/certs/withca/rootCA.crt"
    leader_client_cert_file = "/opt/vault/certs/withca/vault.crt"
    leader_client_key_file = "/opt/vault/certs/withca/vault_nopass.key"
  }
  retry_join {
    leader_api_addr = "https://vault3.tmp.local:8200"
    leader_ca_cert_file = "/opt/vault/certs/withca/rootCA.crt"
    leader_client_cert_file = "/opt/vault/certs/withca/vault.crt"
    leader_client_key_file = "/opt/vault/certs/withca/vault_nopass.key"
  }
}

listener "tcp" {
 address = "vault3.tmp.local:8200"
 cluster_address = "vault3.tmp.local:8201"
 tls_cert_file = "/opt/vault/certs/withca/vault.crt"
 tls_key_file = "/opt/vault/certs/withca/vault_nopass.key"
 tls_client_ca_file = "/opt/vault/certs/withca/rootCA.crt"
 tls_disable = 0
}

ui = true
api_addr = "https://vault3.tmp.local:8200"
cluster_addr = "https://vault3.tmp.local:8201"
cluster_name = "vault-tmp"
log_level = "INFO"
disable_mlock=true
```

### Run

Запускаем это чудо:
```bash
vault server -config=./server-3.hcl
```

Init и unseal

```bash
export VAULT_ADDR='https://vault3.tmp.local:8200'
export VAULT_CACERT=/opt/vault/certs/withca/rootCA.crt
vault operator unseal
# Key from node TWO
```

### Check

```bash
export VAULT_ADDR='https://vault3.tmp.local:8200'
export VAULT_CACERT=/opt/vault/certs/withca/rootCA.crt
vault operator raft list-peers
Node          Address                  State       Voter
----          -------                  -----       -----
oadmintst2    vault2.tmp.local:8201    follower    true
oadmintst3    vault3.tmp.local:8201    leader      true
```
там я переключил leader на vault3 не парьтесь у вас должен быть vault2 лидером


#### !Addon! Join to "Node TWO"

Если вы в конфиге __storage raft__ не написали чет типа
__retry_join__
то можно вручную заджойнится 2мя командами 

Node TWO - это которая первая=)
```bash
vault operator raft join "http://vault2.tmp.local:8200" \
        -leader-ca-cert=@rootCA.crt \
        -leader-client-cert=@vault.crt \
        -leader-client-key=@vault_nopass.key
Key       Value
---       -----
Joined    true
```

Или curl
Делаем payload.json вида
```bash
cat rootCA.crt | awk '{printf $0"\\n"}' # leader_ca_cert
cat vault.crt | awk '{printf $0"\\n"}' # leader_client_cert
cat vault_nopass.key | awk '{printf $0"\\n"}' # leader_client_key
```
Дальше соответсвующий вывод засовываем в payload
```json
{
  "leader_api_addr": "https://vault2.tmp.local:8200",
  "leader_ca_cert": "-----BEGIN CERTIFICATE-----\nMIIDyTCCArGgAwIBAgIUOPbrZV7GBocQ5R7Om5UTHJjWk40wDQYJKoZIhvcNAQEL\nBQAwg==\n-----END CERTIFICATE-----",
  "leader_client_cert": "-----BEGIN CERTIFICATE-----\nMIIEfTCCA2WgAwIBAgIUFs4EIHaueXohyfCR4fBRfMCXguowDQYJKoZIhvcNAQEL\nBQvEsOPmJ0N/I\nSw==\n-----END CERTIFICATE-----",
  "leader_client_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIJKgIBAAKCAgEA0kJnoFo+Pzw5PLEY1RJV+QlfHUNfhbLKI/Ttn8xE4f6MswJh\nBin4kEdHiQbK+bGvW2m4LlilQ==\n-----END RSA PRIVATE KEY-----"
}
```

```bash
curl \
--request POST \
--data @payload.json \
https://vault3.tmp.local:8200/v1/sys/storage/raft/join
```

### Auto unseal

Эт я не думал просто стырил от сюда
https://fdns.github.io/2019-08-16/Basic-Hashicorp-Vault-Autounseal
спасибо добрый человек)

```bash
#!/bin/bash

KEY="hH3wcCstmWZ57j7fHJXlPbft5F5HfJgkGgRG8CuOIsg="
URL="https://vault2.tmp.local:8200"

echo "Starting unsealer"
while true
do
    status=$(curl -k -s $URL/v1/sys/seal-status | jq '.sealed')
    if [ true = "$status" ]
    then
        echo "Unsealing"
        curl -k -s --request PUT --data "{\"key\": \"$KEY\"}" $URL/v1/sys/unseal
    fi
    sleep 10
done
```

P.S Чет задолбался писать пока хватит пожалуй развлечений...