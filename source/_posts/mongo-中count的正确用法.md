---
title: mongo 中count的正确用法
date: 2017-10-13 09:54:35
tags: [python,mongo,count]
---

## 需求背景:
>统计大批量次的某个时间段内的记录数，第一次使用的是count(),但是慢的要死，最后回想起聚合里面可以另辟蹊径统计记录数，下面给出解决方案：
<!--more-->
## 解决方案:

``` python
# -*-coding:utf-8-*-  
#!/usr/bin/env python  
from pymongo import MongoClient
import datetime
import json
db = MongoClient('mongodb://localhost:27017/').tal


start = datetime.datetime.strptime('2017-09-01T','%Y-%m-%dT')
end = datetime.datetime.strptime('2017-10-01T','%Y-%m-%dT')

pipeline = [
     {"$match":{"requestTime":{"$gte":start,"$lt":end}}},
     {"$group":{"_id":{"uri":"null"},"count":{"$sum":1}}}
 ]
ret_val = list(db.sysRequestWrapper.aggregate(pipeline))
ret_ditc = eval(json.dumps(ret_val[0], sort_keys=True))
print ret_ditc['count']

```

