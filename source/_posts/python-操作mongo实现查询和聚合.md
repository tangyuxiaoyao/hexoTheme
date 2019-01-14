---
title: python 操作mongo实现查询和聚合
date: 2017-10-12 09:38:58
tags: [python,mongo]
---

## 需求背景:
>分月分接口统计下我部门所有接口服务每月总计收到的请求数量<!--more-->

## 需求分析
>从需求字面可以理解到分析的维度是**月份**和**接口**，然后再使用**聚合**就应该可以搞定。

## 技术选型
>&ensp;&ensp;&ensp;&ensp;本来觉着使用mongo自带的聚合就可以搞定，但是月份维度的使用则很让我为难，因为时间存储的是**UTC**格式的时间(eg:ISODate("2016-10-18T17:22:04.563Z")),但是分析维度用的只是具体到月份，所以直接使用不是很方便（其实是我没有找到解决方案，但个人理解是这样的两个维度去统计性能也会不理想），所以就考虑到使用容易实现的python来构建json格式的聚合实现，日期则使用遍历和角标结合的解决方案。具体代码如下:

>测试代码test.py(测试服务器以及库和集合的连通性,实现查询数据的功能)
``` python
# -*-coding:utf-8-*-  
#!/usr/bin/env python  
from pymongo import MongoClient


db = MongoClient('mongodb://127.0.0.1:27017/').tal

db_data = db.sysRequestWrapper.find().limit(2)

for data in db_data:
	print data
```
>聚合代码 mongoAggr.py

``` python
# -*-coding:utf-8-*-  
#!/usr/bin/env python  
import json
import datetime
from pymongo import MongoClient
import time

db = MongoClient('mongodb://127.0.0.1:27017/').tal


starts=['2017-01-01T','2017-02-01T','2017-03-01T','2017-04-01T','2017-05-01T','2017-06-01T','2017-07-01T','2017-08-01T','2017-09-01T','2017-10-01T']
for index,day in enumerate(starts):
	if(index == len(starts)-1):
		break
	else:
		start = datetime.datetime.strptime(day,'%Y-%m-%dT')
		end = datetime.datetime.strptime(starts[index+1],'%Y-%m-%dT')
		pipeline = [
		     {"$match":{"requestTime":{"$gte":start,"$lt":end}}},
		     {"$group":{"_id":{"uri":"$uri"},"count":{"$sum":1}}}
		 ]
		res =  list(db.sysRequestWrapper.aggregate(pipeline))
		for item in res:
			retVal = eval(json.dumps(item, sort_keys=True))
			uri = retVal['_id']['uri']
			count = retVal['count']
			print '%s\t%s\t%d' % (day[:7],uri,count)

```
## 总结:
### 难点一:
>&ensp;&ensp;&ensp;&ensp;纠结的时间问题，本来使用的是常规的时间，但是python不像java一样提供的是系统时间，这样导致和服务器的utc时间差八个小时，后来使用了UTC时间来规避这个问题，从而解决了查询不到的问题。

### 难点二
>多个时间区间的问题，本来考虑的是两个for循环搞定，但是灵机一动，考虑到的角标加一的解决方案。

### 注意点:
>结果的格式化、排序，以及str到ditc的转化。

## 笔记
>&ensp;&ensp;&ensp;&ensp;mongo中的date类型以UTC（Coordinated Universal Time）存储，就等于GMT（格林尼治标准时）时间。所以,java读写mongo的Date时，会根据当前系统的时区与GMT进行相互转化。我猜上述转化应该是由java的mongo驱动实现的。比如，在java中，时间2017-09-27 17:57:46.055存入mongo会转化为ISODate("2017-09-27T09:57:46.055Z")，时间少了8小时，这个是由GMT+0800向GMT转化导致的。

>&ensp;&ensp;&ensp;&ensp;而在python中，通常调用datetime.now()或datetime.today()返回当前时间，但是这两个方法只是返回系统时间，不返回时区。而且在python中查询mongo日期类型时，不会进行时区的转化，这就导致查询结果不准（时间大了8小时）。而解决这个问题的方法就是直接以utc格式的时间查询mongo，可以通过datetime.utcnow()返回utc时间，这个方法会根据系统当前时区把时间转化为utc格式，这样就可以查到正确的结果了。