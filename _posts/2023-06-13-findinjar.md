---
layout:     post
title:      "Find in jar..."
subtitle:   " \"---\""
date:       2023-06-13 20:49:00
author:     "ptmp13"
catalog: true
tags:
    - java
---

Предположим нам нужен класс ConfigServiceHelperImpl
```bash
find . -name '*.jar' -type f -exec bash -c 'jar tvf {} | LC_ALL=C grep "ConfigServiceHelperImpl" 1>/dev/null;if [ "$?" = "0" ]; then echo {}; fi' \;
```

Оставлю это тут на потом...
```bash
find . -name '*.jar' -print  -exec /opt/oracle/java/latest/bin/jar tf {} \;|grep lalal
find . -name '*.jar' -print -exec /opt/oracle/java/latest/bin/jar tf {} \; > list_content_jar.txt
```