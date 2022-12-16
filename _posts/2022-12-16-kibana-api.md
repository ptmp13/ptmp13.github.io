---
layout:     post
title:      "Kibana API..."
subtitle:   " \"to be continued...\""
date:       2022-12-14 14:49:00
author:     "ptmp13"
catalog: true
tags:
    - kibana
    - ELK
---

Here some kibana commands for next time...

### Find Dashboard  
Find Dashboard id with title __MegaDashboard__ (output will be _id title_):  
```bash
curl -XGET "http://ololol.local/prod/s/space/api/saved_objects/_find?type=dashboard&search_fields=title&search=Mydash" -u elastic:OLOLOL |jq '.saved_objects[]| "\(.id) \(.attributes.title)" '
```

### Export dashboard 
Export dashboard by id include all subobjects (how to find id in previus step).
```bash
curl -XGET "http://ololol.local/prod/s/space/api/kibana/dashboards/export?dashboard=ea092c10-80f3-11eb-8724-af9cad60a55b" -L -u elastic:app_log_ELK > Mydash.json
```


![to be continued](img/in-post/ELK/tobecontinued.jpg "to be continued...").