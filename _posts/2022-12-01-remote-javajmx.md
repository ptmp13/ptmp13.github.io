---
layout:     post
title:      "Remote connect to jmx using command line client..."
subtitle:   " \"...how to named it...\""
date:       2022-12-01 18:15:00
author:     "ptmp13"
catalog: true
tags:
    - java
    - jar 
    - jmx
---

Its realy stange... but may be you need connect from console to jmx.
#### Java running with following command:

```bash
java -Xms250m -Xmx250m -XX:MaxMetaspaceSize=100m -jar -Dcom.sun.management.jmxremote \
-Dcom.sun.management.jmxremote.port=7000 \
-Dcom.sun.management.jmxremote.ssl=false \
-Dcom.sun.management.jmxremote.authenticate=false \
testjmx.jar
````

Download [jmxterm](https://github.com/jiaqi/jmxterm)

#### Connect to jmx and get Uptime
```bash
java -jar jmxterm-1.0.2-uber.jar -l testserver:7000
$>get -b java.lang:type=Runtime Uptime
#mbean = java.lang:type=Runtime:
Uptime = 316938;
```

help - list avaliable commands


