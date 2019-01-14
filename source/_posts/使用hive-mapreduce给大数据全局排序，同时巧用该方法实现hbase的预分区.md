---
title: 使用hive/mapreduce给大数据全局排序，同时巧用该方法实现hbase的预分区
date: 2018-07-18 09:50:44
tags: [hive,mapreduce,python,row_number,percentile,explode]
---

## 问题描述

>完成排序，打行号，然后根据分位数找到分界点。

<!--more-->

## hive篇

### 造测试数据

> 直接上代码，如下的代码可以仿造出64位的加密字符串，达到模拟某种加密的方式，如此方式造了100w的数据用于测试。

``` python
key = "".join(random.choice("0123456789ABCDEF") for i in range(64))
```
### 导数据

``` sql

use koulb;
CREATE EXTERNAL TABLE t_card_info (key string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
Location '/user/koulingbo/cardInfo';


```

### 排序

``` sql

create table t_card_info_row as 
select row_number() over (order by key) as rn,key from t_card_info;

 number of reducers: 1
```
>众所周知 hive 中的order by 适用于全局排序，所以它只能是给到一个reduce来完成，因为不同的parttion来分区到不同的reduce就决定了只能reduce内部有序，如果你想达到全局排序只能够是一个reduce。

### 分区

>思路：使用分位数函数来定位切割点，然后转置→行转列，与之前的行号join拿到最终的分界点。当然最初你要根据tsv 的大小、snappy压缩比、以及region的大小（根据hfile以及其个数确定）来确定要分成几个region,本文暂定为100个分区来做演示。

``` shell
var="1/100"
for i in {2..99}
do
        var="$var,$i/100"
done
echo $var

hive -S -e "use koulb;create table koulb.t_card_info_res as select k.key from \
(select explode(percentile_approx(rn,array(${var}),2168727))as keyRange \
from koulb.t_card_info_row) r join koulb.t_card_info_row k on floor(r.keyRange)=k.rn;" 

hive -S -e "select * from koulb.t_card_info_res">res


```
### 建表

``` shell
create 'card_quota_test_koulb', {NAME => 'n', VERSIONS => 1, COMPRESSION => 'SNAPPY'},  {SPLITS_FILE => 'res'}
```


## mapreduce部分

==未完待续==