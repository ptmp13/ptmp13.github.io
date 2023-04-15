---
layout:     post
title:      "Linux namespace commands for the next time..."
subtitle:   " \"to be continued...\""
date:       2022-12-20 16:49:00
author:     "ptmp13"
catalog: true
tags:
    - Linux
    - namespace
---
## Get iptables rules from specific pod

1. Get pod pid
```bash
crictl ps
crictl inspect ccb81ebe7a080 # id from previus command
```

We see some info about container 
```json
        "namespaces": [
          {
            "type": "pid"
          },
          {
            "type": "ipc",
            "path": "/proc/3932/ns/ipc"
          },
          {
            "type": "uts",
            "path": "/proc/3932/ns/uts"
          },
          {
            "type": "mount"
          },
          {
            "type": "network",
            "path": "/proc/3932/ns/net"
          }
        ],
```

3. Get namespace for pid 3932

```bash
lsns --output NS,TYPE,PID,COMMAND|grep 3932
4026532826 mnt     3932 /pause
4026532827 ipc     3932 /pause
4026532828 pid     3932 /pause

# OR
ps -e -o pid,netns,ipcns,mntns,pidns,userns,utsns,comm,user |grep 3932

# OR
ll /proc/3932/ns
```

4. Get iptables rules for this

Lol this pid does not have network namespace.
But there must be
Identify namespace:
```bash
# OR for network
ip netns identify 3707
cni-af645ffa-2dcf-10ce-0a8d-b6f970f7c235
```
```bash
ip netns exec cni-af645ffa-2dcf-10ce-0a8d-b6f970f7c235 iptables -t nat -n -L
```

## Some commands

### List all namespace
```bash
lsns
```

### List network namespace
```bash
lsns --type=net
ip netns list
```

### List namespace for pid
```bash
lsns --task pid
```

### List columns
```bash
lsns --output NS,TYPE,PID,COMMAND
```

### list iptables by namespace
```bash
ip netns exec cni-71108cea-44fb-7d93-8e48-cbcaf8f0d678 iptables -t nat -n -L
```
###

### Check packet from namespace
```bash
ip netns exec myns1 nc 127.0.0.1 8083 -v -G3 -w3
```

### List route namespace
```bash
ip netns exec myns1 route -n
ip netns exec cni-71108cea-44fb-7d93-8e48-cbcaf8f0d678 ip route list
```

### Ping from namespace
```bash
ip netns exec myns1 ping 10.1.1.3
```

P.S
[How to Create a Network Namespace and add iptables rules and Test it](https://fosshelp.blogspot.com/2014/07/create-network-namespace-iptables-rules.html)
[Good things](https://windsock.io/a-basic-container/)

