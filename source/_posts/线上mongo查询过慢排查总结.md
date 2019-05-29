---
title: 线上mongo查询过慢排查总结
tags: [mongo,explain]
---


## 现象描述

>日志监控系统提示mongo查询耗时过长，提示信息如下

``` shell
【app】银联智惠监控中心
项目: goblin#01->mongo
主机: lrma02
时间: 2019-02-27 15:30:00
详情: mongo响应时间超过阈值,1分钟内响应时间超过1000ms占比大于10.0%,当前占比:12.20%,总请求次数:164
```
<!--more-->

## 排查思路
> 针对以上提示，可以定位到中间件为mongo，相关人员第一时间要通知到运维查看mongo的操作日志，从而在后台定位到查询耗时过长的是那张表以及查询条件是什么？日志表象如下:

``` java

2019-02-27T15:21:07.038+0800 I COMMAND  [conn25570] command clearing.fee_detail command: find { find: "fee_detail", filter: { account: "F1130004", interfacePath: "/index/personal", smartOrderId: "91dce17a-bd66-4eeb-959c-57f961802a72" }, limit: 1, singleBatch: true } planSummary: IXSCAN { account: 1, interface_path: 1, batch: 1 } keysExamined:351130 docsExamined:351130 cursorExhausted:1 numYields:2750 nreturned:0 reslen:107 locks:{ Global: { acquireCount: { r: 5502 } }, Database: { acquireCount: { r: 2751 } }, Collection: { acquireCount: { r: 2751 } } } protocol:op_query 1750ms

```
>从操作日志可定位到表名:fee_detail,查询的条件涉及到的字段: { account: "F1130004", interfacePath: "/index/personal", smartOrderId: "91dce17a-bd66-4eeb-959c-57f961802a72" },接下来需要做的就是使用该条件去查询该表并打印它的执行计划，在执行计划中查看
>命令: db.table.find({ account: "F1130004", interfacePath: "/index/personal", smartOrderId: "91dce17a-bd66-4eeb-959c-57f961802a72" }).explain()

``` shell
 db.table.find({"col":"CYHS1301942"}).explain()
{
    "queryPlanner" : {
        "plannerVersion" : 1,
        "namespace" : "db.table",    #集合
        "indexFilterSet" : false,
        "parsedQuery" : {
            "b" : {
                "$eq" : "CYHS1301942"
            }
        },
        "winningPlan" : {
            "stage" : "FETCH",
            "inputStage" : {
                "stage" : "IXSCAN",     #索引扫描，COLLSCAN表示全表扫描。
                "keyPattern" : {
                    "account" : 1,
                    "interfacePath" : 1
					"smartOrderId" : 1
                },
                "indexName" : "account_1_interfacePath_1_smartOrderId_1", #索引名
                "isMultiKey" : false,
                "direction" : "forward",
                "indexBounds" : {
                    "account" : [
                        "[\"CYHS1301942\", \"CYHS1301942\"]"
                    ],
                    "interfacePath" : [
                        "[MinKey, MaxKey]"
                    ],
					"smartOrderId" : [
					 "[\"91dce17a-bd66-4eeb-959c-57f961802a72\", \"91dce17a-bd66-4eeb-959c-57f961802a73\"]"
					]
                }
            }
        },
        "rejectedPlans" : [ ]
    },
    "serverInfo" : {
        "host" : "mongo1",
        "port" : 27017,
        "version" : "3.0.4",
        "gitVersion" : "0481c958daeb2969800511e7475dc66986fa9ed5"
    },
    "ok" : 1
}
```

>从执行计划中从"stage" : "IXSCAN"该步骤中keyPattern可以看出具体使用的索引内容从而定位到用的复合索引是不是传输所用的索引。下午发现的问题是因为使用的索引并非建立的索引导致的查询缓慢。

>也可以使用查看所有索引的方式来直接定位是不是所用到的索引存在

``` shell
  db.fee_detail.getIndexes()
[
     {
         "v" : 2,
         "key" : {
             "_id" : 1
         },
         "name" : "_id_",
         "ns" : "clearing.fee_detail"
     },
     {
         "v" : 2,
         "key" : {
             "account" : 1,
             "interface_path" : 1,
             "batch" : 1
         },
         "name" : "account_path_batch_index",
         "ns" : "clearing.fee_detail"
     },
     {
         "v" : 2,
         "key" : {
             "createDate" : 1
         },
         "name" : "createDate_expire_index",
         "ns" : "clearing.fee_detail",
         "expireAfterSeconds" : NumberLong(31536000)
     }
]
```
>从上可以看出并没有找到要用到的索引。
## 解决方案

>定位到问题之后根据查询条件以及业务需要手动地在该表中给该三个字段添加索引。然后观察日志，查询缓慢的问题得到解决。
+ 添加索引的脚本

``` shell
db.fee_detail.ensureIndex({account: 1, interface_path: 1, smartOrderId:1});
```

## 方案优化
+ 待考量的三种方案


``` shell
1.account,interface_path,smartOrderId;account,interface_path,batch 两个复合索引
2.account,interface_path;batch;smartOrderId 三个索引
3.account;interface_path 两个索引
```

## 方案跟踪


+ 方案一:

``` shell
db.fee_detail.ensureIndex({account: 1, interface_path: 1, smartOrderId:1});
db.fee_detail.ensureIndex({account: 1, interface_path: 1, batch:1});
```

+ 方案二

``` shell
db.fee_detail.ensureIndex({account: 1, interface_path: 1});
db.fee_detail.ensureIndex({batch:1});
db.fee_detail.ensureIndex({smartOrderId:1});
```

+ 方案三：

``` shell
db.fee_detail.ensureIndex({account:1});
db.fee_detail.ensureIndex({interface_path:1});
```

+ 测试步骤

----

 1. 准备mongo存量数据,分别100W,500W,1000W。
 2. 并发请求接口，造４要素的数据确保４要素不会重复，这样每次都会调用第三方不走缓存＝＞每次都会insert和update mongo库。
 3. 获取系统的日志记录的插入mongo库和更新mongo库的时间解析，计算插入和更新mongo库的平均响应时间。

``` java
        　　com.unionpaysmart.goblin.service.MongoService[83]:middleware_opt|mongo|5573|success|插入mongo 成功日志=>5573
```

----
+ 比较结果见下图


![mongo 执行比较](/ITWO/assets/mongo-data.png)


## 进一步跟踪

>既然第二种和第三种查询速率不分伯仲，望再帮忙看下，这两种分别建立的索引占的空间大小。
>命令如下:

``` shell
db.sys_request_info.stats(1024*1024)
查看totalIndexSize的大小 单位是M 
```

+ 结果如下:

``` shell
２种方案创建的索引大小如下，请参考：

方案一:
db.fee_detail.ensureIndex({account: 1, interface_path: 1, smartOrderId:1});
db.fee_detail.ensureIndex({account: 1, interface_path: 1, batch:1});
PRIMARY>  db.fee_detail.stats(1024*1024)
{
	"totalIndexSize" : 2100,
	"indexSizes" : {
		"_id_" : 573,
		"createDate_expire_index" : 492,
		"account_1_interface_path_1_smartOrderId_1" : 648,
		"account_1_interface_path_1_batch_1" : 386
	},
	"ok" : 1
}

方案二:
db.fee_detail.ensureIndex({account: 1, interface_path: 1});
db.fee_detail.ensureIndex({batch:1});
db.fee_detail.ensureIndex({smartOrderId:1});
PRIMARY>  db.fee_detail.stats(1024*1024)
{
	"totalIndexSize" : 2145,
	"indexSizes" : {
		"_id_" : 573,
		"createDate_expire_index" : 492,
		"account_1_interface_path_1" : 267,
		"batch_1" : 277,
		"smartOrderId_1" : 535
	},
	"ok" : 1
}

```