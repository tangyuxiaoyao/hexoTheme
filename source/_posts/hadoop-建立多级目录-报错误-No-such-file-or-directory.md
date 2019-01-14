---
title: hadoop 建立多级目录　报错误　　No such file or directory
date: 2017-11-02 20:22:29
tags: [hadoop,no,such,file]
---

## 需求背景

>在shell脚本想建立多层ｈｄｆｓ目录时，报错。
在HDFS中创建多级目录，然而总是报错：
<!--more-->
``` shell
mkdir: `/user/a/bb': No such file or directory。
```
## 解决方案

>在StackOverflow上面某牛说是命令本身有问题，应该是 $HADOOP_HOME/bin/hadoop fs -mkdir -p /user/hive/warehouse，使用该命令，不再报错，问题完美解决！
