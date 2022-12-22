---
layout:     post
title:      "Elasticsearch Calculate the Storage Size of Data streams..."
subtitle:   " \"...let's find it using awk grep sed...\""
date:       2022-11-30 21:15:00
author:     "ptmp13"
catalog: true
tags:
    - elasticsearch
    - ELK 
    - kibana
    - bash
---

Ok... we have ELK with some nodes.
Sometimes, when we work with ELK, we have some questions:
1. *What datastream mostly disk usage in cluster?*
2. *What datastream mostly disk usage on this node?*


I propose to answer on these questions using bash

## 1. What datastream mostly disk usage in cluster?

Next command shows datastreams sorted by ascending. (output disk usage in Gb)

```bash
export elasticURL=https://olol.elasticsearch.com:9200
export elasticCRED=elastic:passwd
curl -N -k -XGET "${elasticURL}/_cat/shards?h=i,sto,n&bytes=b" \
-u ${elasticCRED}|tac|grep "\.ds-"|\
gawk -v OFMT='%.5f' \
'{sub(/-[^-]*-[^-]*$/,"",$1); a[$1]+=$2;} \
END {for(i in a){print sprintf("%.15f", a[i]/1024/1024/1024/1024*1e3),i }}'|\
sort -k 1 -n
```


## 2. What datastream mostly using space on this node?

Ok now this same as overall nodes, but we add "grep": 
```bash
grep ${elasticNode}
```

```bash
export elasticURL=https://olol.elasticsearch.com:9200
export elasticCRED=elastic:passwd
export elasticNode=data-1
curl -N -k -XGET "${elasticURL}/_cat/shards?h=i,sto,n&bytes=b' \
-u ${elasticCRED}|tac|grep ${elasticNode}|grep "\.ds-"|\
gawk -v OFMT='%.5f' \
'{sub(/-[^-]*-[^-]*$/,"",$1); a[$1]+=$2;} \
END {for(i in a){print sprintf("%.15f", a[i]/1024/1024/1024*1e3),i }}'|\
sort -k 1 -n
````

Sometimes you must add grep -v "node3" after first grep
its mean exclude __node3__, becouse when shard move from one node to another in query "__\_cat/shards__" we will see two nodes.

#### P.S After you find some datasteam you can use API for check fields etc...

```bash
curl -XPOST ${elasticURL}/my_datastream/_disk_usage?run_expensive_tasks=true
```
[elasticsearch docs](https://www.elastic.co/guide/en/elasticsearch/reference/master/indices-disk-usage.html)
