---
title: hadoop  streaming 二次排序join的实现
date: 2017-11-02 20:46:25
tags: [hadoop,streaming,join]
---

## 需求背景
>提供一批卡号，去提取这批卡号的流水(lzo格式的交易流水)
>一个典型的Map-Reduce过程包 括：Input->Map->Partition->Reduce->Output。
<!--more-->
## 模拟实现逻辑

![scrapy 流程图](/ITWO/assets/mapreduce03.jpg)

## 配置文件如下

``` bash
#!/bin/bash
hadoop fs -test -d  ${2}
if [$? -eq 0]
then
	hadoop fs -rmr card_match_2/output
fi

hadoop jar ~/koulb/softjar/hadoop-streaming-2.0.0-mr1-cdh4.7.0.jar \

-D map.output.key.field.separator=_ \

-D num.key.fields.for.partition=1 \

-D stream.map.input.ignoreKey=true \

-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \

-inputformat com.hadoop.mapred.DeprecatedLzoTextInputFormat \

-input ${1}\

-output ${2} \

-file map.py \

-file red.py \

-mapper "python map.py" \

-reducer "python red.py" \

-jobconf mapred.job.priority=VERY_HIGH \

-jobconf mapred.reduce.tasks=40 \

-jobconf mapred.job.name="card_match"


```


## 参数普及
>一个典型的Map-Reduce过程包 括：Input->Map->Partition->Reduce->Output。
``` shell
hadoop streaming /    
-D stream.map.output.field.separator=, /    
-D stream.num.map.output.key.fields=4 /    
-D map.output.key.field.separator=, /    
-D num.key.fields.for.partition=2 /    
-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner /    
```
>stream.map.output.field.separator 指定map输出时使用的分隔符
>stream.num.map.output.key.fields 指定用上面的分隔符分割的前四部分作为map输出的key
>map.output.key.field.separator 指定二次排序的key内部使用的分隔符
>num.key.fields.for.partition 指定使用上面设置的分隔符,切割key出的前两部分作为二次排序的依据
>org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner 指定分区用的分割器
>如果不指定默认的分割器为HashedPartitioner
>以上实现的效果为:map输出时指定以","为分割符的字段的前四个key,其余部分作为value,这时,map输出的key被3个","分割成4部分,前2部分用于Partition,前2部分相同的key切分到同一个reducer,因此reduce内部的key排序时前2部分相同的key实际上是对后面2部分排序,这样就相当于前2部分作为排序主键,后2部分作为排序的辅键.
## 疑惑
>在使用默认的分隔符"\t"去实现二次排序,一定不要指定stream分割符和partitioner分隔符,否则会导致二次排序失效. 类似 cut -d "\t"  ??,使用java mapredue时如果直接用"\t",就会出现结果文件里面分隔符为\t字符,是否也是同样道理?

``` shell
    -jobconf stream.num.map.output.key.fields=4 \
    -jobconf num.key.fields.for.partition=3 \
    -jobconf mapred.reduce.tasks=2 \
    -input koulb/aa \
    -output $1 \
    -mapper "cat" \
    -reducer "cat" \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner

```
