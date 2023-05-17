---
layout:     post
title:      "Rsyslog -> Kafka <- Logstash"
subtitle:   " \"...\""
date:       2023-05-17 14:44:00
author:     "ptmp13"
catalog: true
tags:
    - Kafka
    - Logstash
    - Rsyslog
---

Кафка для теста:

```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
bin/kafka-server-start.sh config/kraft/server.properties
```

В файле _config/kraft/server.properties_ ставим __num.partitions=4__

В /etc/rsyslog/apache.conf закидываем
(broker надо поправить на нужное).

```bash
module(load="omkafka")

$template logstash-httpd, "%msg%\n"

main_queue(
  queue.workerthreads="8"
  queue.dequeueBatchSize="100"
  queue.size="10000"
)

if ( $syslogfacility-text == "local6" ) then
{
  action(
    broker=["ololol.local:9092"]
    type="omkafka"
    topic="httpdtest-kafka"
    template="logstash-httpd"
    partitions.auto="on"
  )
}
```

__partitions.auto="on"__ нужно в случае если у нас больше одной партиции

Проверяем че нить пуляем туда
```bash
echo "ololol alalal"|/usr/bin/logger -t apache -p local6.info
```

Видим что все лежит в разных партициях ураган...
![img](/img/in-post/Kafka/multipart.png)

Ну дальше короче можно читать logstash эт потом допишу если че...
