---
layout:     post
rewards: false
title:  es stat
categories:
    - es
tags:
    - big data

---

```python3
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import logging
import sys
import time
from pprint import pprint
import requests

ES_URL_STAT = 'http://127.0.0.1:9200/_stats'
TIME_DURATION = 60
compare_data = {}

while True:
    resp = requests.get(ES_URL_STAT)
    resp_data = resp.json()
    indices = resp_data['indices']
    for index_name, index_data in indices.items():
        index_data = index_data['primaries']
        compare_data[index_name] = {
            'index': index_data['indexing']['index_total'] / ((index_data['indexing']['index_time_in_millis']+1) / 1000),
            'search_query': index_data['search']['query_total'] / ((index_data['search']['query_time_in_millis']+1) / 1000),
            'search_fetch': index_data['search']['fetch_total'] / ((index_data['search']['fetch_time_in_millis']+1) / 1000),
            'refresh': index_data['refresh']['total'] / ((index_data['refresh']['total_time_in_millis']+1) / 1000),
        }

    print(compare_data)
    print()
    sys.stdout.flush()
    time.sleep(TIME_DURATION)
```

[Elasticsearch Index Monitoring(索引监控)之Index Stats API详解](https://cloud.tencent.com/developer/article/1444025)

