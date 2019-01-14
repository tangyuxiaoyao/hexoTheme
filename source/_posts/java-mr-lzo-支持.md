---
title: java mr lzo 支持
date: 2018-09-13 11:08:56
tags: [java,mapreduce,lzo]
---

## 背景描述

> lzo格式有很大的压缩比,相对于Hdfs文件而言有很大优势,支持分片,默认是不支持splitable的，需要为其添加索引文件，才能支持多个map并行对lzo文件进行处理.先阶段来说为了节约存储空间,lzo格式的文件存储随处可见,所以在mapreduce中怎么读取和存储lzo文件就很有必要了解下.

<!--more-->


## 技术实现



>如果希望mr输出的是lzo格式的文件，添加下面的语句

``` java
FileOutputFormat.setCompressOutput(job, true);
FileOutputFormat.setOutputCompressorClass(job, LzopCodec.class);
int result = job.waitForCompletion(true) ? 0 : 1;
//上面的语句执行完成后，会生成最后的输出文件，需要在此基础上添加lzo的索引
LzoIndexer lzoIndexer = new LzoIndexer(conf);
lzoIndexer.index(new Path(args[1]));
```
>如果已经存在lzo文件，但没有添加索引，可以采用下面的方法，在输入路径的文件上上添加lzo索引
``` shell
hadoop jar $HADOOP_HOME/lib/hadoop-lzo-0.4.17.jar com.hadoop.compression.lzo.LzoIndexer hdf://inputpath
```


