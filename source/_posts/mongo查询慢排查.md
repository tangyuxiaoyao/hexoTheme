---
title: mongo查询慢排查
date: 2018-05-23 14:11:34
tags: [mongo,shell,dbKoda]
---

## 问题描述
>线上建模结果缓存在mongo中，之前由于数据量较少，进来一段时间由于老系统业务迁移过来，数据量也上来了，随之而来暴露出来一个问题.

<!--more-->

>查询很慢，但是只是个别客户的账号有问题，起初以为是客户建模的数据（客户部分代码有异常）有问题导致，但是从日志来看也有正常返回的数据但是也会有超时，后来把该客户的数据导入到开发集群来排查问题。

## 解决思路

>之前接口层面建的索引执行计划给出执行过程

![之前接口层面建的索引执行计划给出执行过程]( /ITWO/assets/index_01.png)

>分解图示：
>Explain部分分为

 1. IXSCAN（检索索引）
 	 耗费250ms,排查（examined）文档965978个，该步骤返回文档965978个，用到的索引是account_1_analysisId_1，该索引匹配到的数据965978.
 2. FETCH（拉取数据）
	 耗时660ms,排查（examined）文档965978个，该步骤返回文档１个，遍历加判断最终拿到想要的结果，这步耗时也是最多的一步。
 3. KEEP_MUTATIONS（暂时不知道是干嘛的）

>其实仔细看已经看出端倪了，这边提示的是Index account_1_analysisId_1 was used to find matching values for account,**analysisId**，还有looking for these criteria: {**"analysis_id".**:{"$eq":"bc70380f-47b0-40f9-9e24-f54d61ce8bd6"}}，复合索引中的字段并不是我们检索的mongo字段，后来翻查代码，

``` java
@Document(collection = "bill_analysis_info")
@CompoundIndexes({ @CompoundIndex(name = "query_index", def = "{account : 1, analysisId : 1}") })
public class BillAnalysisInfo implements Serializable {……}
```
>找到如上的根源。

## 解决办法
>手动建立针对这两个业务字段的复合索引

``` shell
db.bill_analysis_info.ensureIndex({"account":1,"analysis_id":1})
```
>再次查看同样查询的执行计划

![重建索引执行计划给出执行过程]( /ITWO/assets/index_02.png)

>**效果很明显**

## 写在后面

>mongo中原本以为会对建立索引过程中的字段是否为该集合的字段做判断，但此次排查下来明显没有，引以为戒。
>细节，认真。
>可视化工具用到的是dbKoda，要求mongo3.0版本以上。