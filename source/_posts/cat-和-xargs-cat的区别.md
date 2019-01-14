---
title: cat 和　xargs cat的区别
date: 2017-10-18 10:26:31
tags: [shell,cat,xargs]
---

## 需求分析
>在hadoop官方文档上有看到这样的使用
<!--more-->
``` shell
 $HADOOP_HOME/bin/hadoop  jar $HADOOP_HOME/hadoop-streaming.jar \
                  -archives 'hdfs://hadoop-nn1.example.com/user/me/samples/cachefile/cachedir.jar' \  
                  -D mapred.map.tasks=1 \
                  -D mapred.reduce.tasks=1 \ 
                  -D mapred.job.name="Experiment" \
                  -input "/user/me/samples/cachefile/input.txt"  \
                  -output "/user/me/samples/cachefile/out" \  
                  -mapper "xargs cat"  \
                  -reducer "cat"  
```
>其中 mapper中用的可执行命令是xargs cat而redcuer用的是cat,这样的区别在哪里？

## 技术分析
>管道是实现“将前面的标准输出作为后面的标准输入”
>xargs是实现“将标准输入作为命令的参数”

>你可以试试运行：

``` bash
echo "--help"|cat
echo "--help"|xargs cat
```


