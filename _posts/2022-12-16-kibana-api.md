---
layout:     post
title:      "Kibana/Elasticsearch API..."
subtitle:   " \"to be continued...\""
date:       2022-12-14 14:49:00
author:     "ptmp13"
catalog: true
tags:
    - kibana
    - ELK
---

# Here some kibana commands for next time...

### Find Dashboard  
Find Dashboard id with title __MegaDashboard__ (output will be _id title_):  
```bash
curl -XGET "http://ololol.local/prod/s/space/api/saved_objects/_find?type=dashboard&search_fields=title&search=Mydash" \
-u elastic:OLOLOL |jq '.saved_objects[]| "\(.id) \(.attributes.title)" '
```

### Export dashboard 
Export dashboard by id include all subobjects (how to find id in previus step).
```bash
curl -XGET "http://ololol.local/prod/s/space/api/kibana/dashboards/export?dashboard=ea092c10-80f3-11eb-8724-af9cad60a55b" \
-L -u elastic:OOLOL > Mydash.json
```

### Import Dashboard

for import dashboard to space:
```bash
curl -f -XPOST -u elastic:OLOLOL \
-H 'Content-Type: application/json' \
-H 'kbn-xsrf: this_is_required_header' \
'http://ollool.local/prod/s/space/api/kibana/dashboards/import' \
-T ./atatat.json
```

### Search all users that have role

Find all users that have role MYROLE
```bash
curl -XGET http://ololol.local/elasticsearch/_security/user 
-u elastic:OLOLOL | jq '.[] | select(.roles[]=="MYROLE")
```

For only username:
```bash
curl -XGET http://ololol.local/elasticsearch/_security/user 
-u elastic:OLOLOL | jq -c '.[] | select(.roles[]=="MYROLE") | .username'
```

### Add role if its not exist


Check script before use it=)))
```bash
#!/bin/bash
export ELKuser="elastic"
export ELKpass="OLOLOL"
export ELKURL="http://lolo.local/elasticsearch"
# Get users with role MYROLE and save to file listusers.txt
curl -XGET ${ELKURL}/_security/user -u ${ELKuser}:${ELKpass} | jq -c '.[] | select(.roles[]=="MYROLE")' > listusers.txt

while read line
do
        updateuser=$(echo $line | jq -c '.roles += ["TESTROLE","TESTROLE2","TESTROLE3"] | .roles')
        username=$(echo $line | jq -c '.roles += ["TESTROLE","TESTROLE2","TESTROLE3"] | .username'|tr -d \")
        jqroles=$(echo "{ \"roles\": $updateuser }")
        echo "user: $username"
        echo "{ \"roles\": $updateuser }"
        curl -H "Content-Type: application/json" -XPOST -u ${ELKuser}:${ELKpass} "${ELKURL}/_security/user/${username}" -d "$jqroles"

done <listusers.txt

```

![to be continued](../img/in-post/ELK/tobecontinued.jpg "to be continued...").