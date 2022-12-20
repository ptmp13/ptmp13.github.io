---
layout:     post
title:      "Linux socket uptime..."
subtitle:   " \"to be continued...\""
date:       2022-12-14 14:49:00
author:     "ptmp13"
catalog: true
tags:
    - linux
---

Problem view like this:

> 14-Dec-2022 14:13:15.729 SEVERE [http-nio-10.64.130.81-8080-Acceptor-0] org.apache.tomcat.util.net.Acceptor.run Socket accept failed
 java.io.IOException: Too many open files
        at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)


Ok check fd this process
```bash
]$ ls -l /proc/26700/fd|wc -l
3989
```
```bash
]$ ls -l /proc/26700/fd
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 990 -> socket:[343227304]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 991 -> socket:[344107160]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 992 -> socket:[343195983]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 993 -> socket:[343190714]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 994 -> socket:[344436574]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 995 -> socket:[344400363]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 996 -> socket:[344413309]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 997 -> socket:[344103141]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 998 -> socket:[344099280]
lrwx------ 1 tomcat tomcat 64 Nov  9 11:01 999 -> socket:[344128665]
```

Ok we see many sockets open.

Same check using netstat
```bash
]$ netstat -nap|grep 26700|wc -l
3902
```
```bash
tcp6       0      0 1.1.1.1:47870      1.1.1.2:22        ESTABLISHED 26700/java
tcp6       0      0 1.1.1.1:39882      1.1.1.2:22        ESTABLISHED 26700/java
tcp6       0      0 1.1.1.1:42212      1.1.1.2:22        ESTABLISHED 26700/java
tcp6       0      0 1.1.1.1:52267      1.1.1.2:22        ESTABLISHED 26700/java
```


This strange things from [theme](https://superuser.com/questions/565991/how-to-determine-the-socket-connection-up-time-on-linux)
```bash
find /proc/26700/fd -lname "socket:*" -printf "\n%AD %AT %p"|sort -k1.8n -k1.1nr -k1|more
```

Script (for the next time):
```bash
#!/bin/bash
PID=${1}
CONNECTIONS=$(netstat -nape|grep ${PID}|grep "ESTABLISHED" | awk '{print $5","$8","$9}')
echo "total connections for pid ${PID}: $(echo ${CONNECTIONS} | wc -w)"
for cdevice in ${CONNECTIONS}; do
    info=$(echo ${cdevice}|awk -F',' '{print $1}')
    device=$(echo ${cdevice}|awk -F',' '{print $2}')
    timestamp=$(find /proc/${PID}/fd -lname "socket:\[${device}\]" -printf %T@ 2> /dev/null);
    # compute the time difference
    LANG=C printf '%s ( %.2f s ago)\n' "${info} $(date -d @$timestamp)" $(bc <<<"$(date +%s.%N) - $timestamp");
    echo;
done
```

Run:
```bash
./checktime.sh 26700
```

Ok we see something like this:
>total connections for pid 31096: 1  
10.64.130.81:22 Wed Dec 14 18:00:16 MSK 2022 ( 184.02 s ago)  

But i know that problem occured two hours ago. And if we closer look at the script:
```bash
 -printf %T@ 
```
it means  
%T - _File's last modification time._
@ - _seconds since Jan. 1, 1970, 00:00 GMT, with fractional part._

But we need to know when socket created, not _modification time_... for example i start ssh connection at 18:34:00
```bash
> stat /proc/11835/fd/3
  File: /proc/11835/fd/3 -> socket:[34517616]
  Size: 64              Blocks: 0          IO Block: 1024   symbolic link
Device: 0,20    Inode: 34907466    Links: 1
Access: (0700/lrwx------)  Uid: ( 1000/   peter)   Gid: ( 1000/   peter)
Access: 2022-12-14 19:59:29.887846821 +0300
Modify: 2022-12-14 19:59:29.887846821 +0300
Change: 2022-12-14 19:59:29.887846821 +0300
```

We can use ss for check:

>lastsnd:<lastsnd>  
    how long time since- the last packet sent, the unit is millisecond.    
lastrcv:<lastrcv>  
    how long time since the last packet received, the unit is millisecond.    
lastack:<lastack>  
    how long time since the last ack received, the unit is millisecond  

```bash
ss -Hnapei -O -T dst :22
```

And we can see thread life time
```bash
ps -eo pid,tid,class,rtprio,ni,pri,psr,pcpu,stat,wchan:14,comm,etime|grep 16953
```

After restart tomcat all sockets go to TIME_WAIT for
```bash
cat /proc/sys/net/ipv4/tcp_fin_timeout
```

to be continued...

Links:
[https://superuser.com/questions/565991/how-to-determine-the-socket-connection-up-time-on-linux](https://superuser.com/questions/565991/how-to-determine-the-socket-connection-up-time-on-linux)

P.S
ss -nap dst :22
--info
-K 