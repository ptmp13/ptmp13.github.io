---
layout:     post
title:      "Gitlab Cache..."
subtitle:   " \"... some thing about cache ... \""
date:       2024-05-13 12:10:00
author:     "ptmp13"
catalog: true
tags:
    - gitlab
    - minio
---

> Ну че типа разберемся с кэшом...

В качестве основы возьмем статью [GitLab CI speeding up your pipeline with caching](https://dev.to/jnmoal/gitlab-ci-speeding-up-your-pipeline-with-caching-5400)

Чуток модифицируем это дело вообщем поехали...

## Пару слов о конфигурации 

Нам понадобится __Gitlab Runner__ + __minio__ это нужно для использования _fallback_keys_ в конфигурации кэша (ниже будет подробнее).  
В доке к сожалению подробностей нет но можно почитать тут [Fallback cache key only works for distributed cache](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/27537).  
Тут писать что-то о настройке __gitlab-runer + minio__ мы не будем просто приведу конфиг

```toml
listen_address = "runner.local:9252"
concurrent = 20
check_interval = 2

[session_server]
  session_timeout = 1800

[[runners]]
  name = "runner"
  url = "https://gitlab.local/"
  limit = 20
  request_concurrency = 10
  id = 521
  token = "xdfsFGcvdfsvffdsF"
  token_obtained_at = 2023-09-19T14:51:02Z
  token_expires_at = 0001-01-01T00:00:00Z
  tls-ca-file = "/media/data/gitlab-runner/config/certs/gitlab.pem"
  executor = "shell"
  environment = ["GIT_SSL_NO_VERIFY=1"]
  [runners.cache]
    MaxUploadedArchiveSize = 0
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      AccessKey = "xzgxgdfgdfg"
      SecretKey = "dxdgdxgGfxzdhf"
      BucketName = "runner"
      Insecure = true
      ServerAddress = "runner.local:80"
```

## Тестовый конфиг для GitLab CI

Тут стоит у помянуть что у нас будут "псевдошаги" те по факту будем просто писать и выводить файл.
В краце опишем что мы хотим

Скажем у нас есть 2 ветки:  
* master
* dev

### Stage + Variables

Фиг знает че тут сказать вообще то нечего... ну разве что по переменным.  

__DEFAULT_CACHE_KEY_PREFIX__ - эт типа для fallback кэша т.е. если мы не будем собирать кэш то от куда его брать.  
__CACHE_KEY_PREFIX__ - тут просто префикс для кэша.  
__DEPS_FOLDER__ - директория которую будем кэшировать.  

```yaml
stages:
  - prepare
  - build
  - test

variables:
  DEFAULT_CACHE_KEY_PREFIX: ${CI_DEFAULT_BRANCH}
  CACHE_KEY_PREFIX: ${CI_COMMIT_REF_SLUG}
  DEPS_FOLDER: ${CI_PROJECT_DIR}/.deps
```

### Когда собирать кэш?

Собирать кэш будем в 2х случаях.  
1. В случае билда из ветки мастер  
2. Если мы билдим любую другую ветку при этом у нас файл my-referent-file.lock изменился в сравнии с файлом из ветки master [doc](https://docs.gitlab.com/ee/ci/yaml/#ruleschangescompare_to).  

За это дело у нас будут отвечать следующий кусок конфига  

```yaml
.deps_update_rules:
  rules:
    - if: $CI_COMMIT_BRANCH != "master"
      changes:
        paths:
          - my-referent-file.lock
        compare_to: refs/heads/master
    - if: $CI_COMMIT_BRANCH == "master"
      when: always
    - if: $CI_COMMIT_TAG != null
      when: always
```

### Anchor для сборки кэша

Тут у нас пару кэшей будет формироваться.  
1. ${CACHE_KEY_PREFIX}-deps-dev
2. ${CACHE_KEY_PREFIX}-deps-run

(Их 2 не потому что тут что-то хитрожопое просто в изначальной статье их зачем то сделали 2, ну и для наглядности пойдет)

```yaml
.deps_dev_cache:
  cache: &deps_dev_cache
    key: ${CACHE_KEY_PREFIX}-deps-dev
    fallback_keys:
      - ${DEFAULT_CACHE_KEY_PREFIX}-deps-dev
    paths:
      - ${DEPS_FOLDER}
    policy: pull

.deps_run_cache:
  cache: &deps_run_cache
    key: ${CACHE_KEY_PREFIX}-deps-run
    fallback_keys:
      - ${DEFAULT_CACHE_KEY_PREFIX}-deps-run
    paths:
      - ${DEPS_FOLDER}
    policy: pull
```

Собственно че тут написанно про сами anchors (&deps_dev_cache/&deps_run_cache) писать не буду можно почитать [тут](https://docs.gitlab.com/ee/ci/yaml/yaml_optimization.html#anchors). Тут просто нужно знать что anchors пишутся для того чтобы можно бы ло переиспользовать данный контент где то ниже.

Соответственно тут у нас "контент" для формирования кэша: 
* __key: ${CACHE_KEY_PREFIX}-deps-dev__ - название кэша
* __fallback_keys: - ${DEFAULT_CACHE_KEY_PREFIX}-deps-dev__ - если не найден кэш ${CACHE_KEY_PREFIX}-deps-dev то попробовать взять кэш из __fallback_keys__
* paths: ${DEPS_FOLDER} - путь который будем кэшировать
* policy - забирать пулять вообщем типа че делать с кэшом, данное свойство будем переопредлять по ходу написания конфига.

### Сбор кэша

```yaml
dev_dependencies:
  stage: prepare
  script:
    - echo "I am a dependency" > ./.deps/deps.txt
    - echo "I am a dev dependency" >> ./.deps/deps.txt
  cache:
    <<: *deps_dev_cache
    policy: pull-push
  rules: !reference [.deps_update_rules, rules]

run_dependencies:
  stage: prepare
  script:
    - echo "I am a dependency" > ./.deps/deps.txt
  cache:
    <<: *deps_run_cache
    policy: pull-push
  rules: !reference [.deps_update_rules, rules]
```

Тут мы формируем 2 кэша путем использования упомянутых выше anchors. В зависимости от rules

* script - ну типа скрипт че тут еще сказать
* cache - тут берем контент из anchor deps_dev_cache
* policy - тут подменяем policy взятое из anchors на ту которая позваляет так же засывывать кэш

### Проверка кэша

```yaml
test-app:
  stage: test
  script:
    - cat ./.deps/deps.txt
  cache:
    <<: *deps_dev_cache

package-app:
  stage: build
  script:
    - cat ./.deps/deps.txt
  cache:
    <<: *deps_run_cache
```

тут хз че сказать просто скрипт который выводит содержимое файла при этом опять использует кэши из anchors __deps_dev_cache__ и __deps_run_cache__.  

## Full config

```yaml
stages:
  - prepare
  - build
  - test

variables:
  DEFAULT_CACHE_KEY_PREFIX: ${CI_DEFAULT_BRANCH}
  CACHE_KEY_PREFIX: ${CI_COMMIT_REF_SLUG}
  DEPS_FOLDER: ${CI_PROJECT_DIR}/.deps

.deps_update_rules:
  rules:
    - if: $CI_COMMIT_BRANCH != "master"
      changes:
        paths:
          - my-referent-file.lock
        compare_to: refs/heads/master
    - if: $CI_COMMIT_BRANCH == "master"
      when: always
    - if: $CI_COMMIT_TAG != null
      when: always

.deps_dev_cache:
  cache: &deps_dev_cache
    key: ${CACHE_KEY_PREFIX}-deps-dev
    fallback_keys:
      - ${DEFAULT_CACHE_KEY_PREFIX}-deps-dev
    paths:
      - ${DEPS_FOLDER}
    policy: pull

.deps_run_cache:
  cache: &deps_run_cache
    key: ${CACHE_KEY_PREFIX}-deps-run
    fallback_keys:
      - ${DEFAULT_CACHE_KEY_PREFIX}-deps-run
    paths:
      - ${DEPS_FOLDER}
    policy: pull

dev_dependencies:
  stage: prepare
  script:
    - echo "I am a dependency" > ./.deps/deps.txt
    - echo "I am a dev dependency" >> ./.deps/deps.txt
  cache:
    <<: *deps_dev_cache
    policy: pull-push
  rules: !reference [.deps_update_rules, rules]

run_dependencies:
  stage: prepare
  script:
    - echo "I am a dependency" > ./.deps/deps.txt
  cache:
    <<: *deps_run_cache
    policy: pull-push
  rules: !reference [.deps_update_rules, rules]

test-app:
  stage: test
  script:
    - cat ./.deps/deps.txt
  cache:
    <<: *deps_dev_cache

package-app:
  stage: build
  script:
    - cat ./.deps/deps.txt
  cache:
    <<: *deps_run_cache
```


## Test Runs

Вообщем попробуем позапускать это дело. 
Предположим что на старте у нас есть две абсолютно одинаковые ветки
1) master
2) dev

Нам полюбому надо сначала запустить pipeline по ветке __master__ как минимум для того что бы у нас сформировался кэш. Собственно запускаем видимо что все захорошо

![img](/img/in-post/GitLabCache/SCR-20240513-kixh.png)

Далее запускаем по ветке __dev__ видим что кэши не собирались при этом все шаги отработали хорошо потому что исползован был fallback cache т.е в нашем случае кэш из ветки мастер:
```yaml
    fallback_keys:
      - ${DEFAULT_CACHE_KEY_PREFIX}-deps-dev
```

Видимо что не смотря на то что кэши не собиралась все хорошо
![img](/img/in-post/GitLabCache/SCR-20240513-kkrd.png)

В логе джоба можно в целом увидеть что кэш для dev ветки не найден, и соответствено был звят кэш из ветки master (напомню что для этого надо s3 хранилище)
![img](/img/in-post/GitLabCache/SCR-20240513-kqfy.png)

Попробуем поправить файл _my-referent-file.lock_ в __dev__ ветке собственнааа для того что бы собрался отдельный кэш для этой ветки (я просто добавил туда пару строк)... пушим... смотрим результат:

![img](/img/in-post/GitLabCache/SCR-20240513-ktgd.png)

## P.S.

Всем спасибо!)