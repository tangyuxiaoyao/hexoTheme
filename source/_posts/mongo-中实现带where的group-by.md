---
title: mongo中实现带where的group by
date: 2017-10-09 15:23:24
tags: [mongo,groupby,where]
---
## 需求背景:
>需要在mongo中统计同一个时间段内各个账号的接口调用量．<!--more-->

## 解决思路:
>通常我们在关系型数据库中可以这么解决:

``` sql
select account,count(1) from db.table where requestTime>='2017-10-01' and requestTime <'2017-10-08'
group by account;
```
## 解决方案:
>参照sql语句的思路，然后给出mongo中的解决方案

``` json
db.sysRequestWrapper.aggregate([
    {"$match":{requestTime:{$gte:ISODate("2017-10-01T00:00:00.000Z"),$lt:ISODate("2017-10-19T00:00:00.000Z")}}}, 
    {"$group":{_id:{account:"$account"},"count":{"$sum":1}}}
]);
```
>ps:如果需要实现having count(*)>1 则你需要再在后面追加match,来实现对group by 聚合结果的过滤．
``` json
db.sysRequestWrapper.aggregate([
    {"$match":{requestTime:{$gte:ISODate("2017-10-01T00:00:00.000Z"),$lt:ISODate("2017-10-19T00:00:00.000Z")}}}, 
    {"$group":{_id:{account:"$account"},"count":{"$sum":1}}},
	{"$match":{"count":{"$gt": 1}}}
]);

```
