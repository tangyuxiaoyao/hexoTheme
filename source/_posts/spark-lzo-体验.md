---
title: spark lzo 体验
date: 2017-09-21 17:07:52
tags: [spark,lzo]
---

## 需求背景：

>项目中需要读取lzo文件并且提取其中的几个字段放到表中，与另一个文件join，建立映射文件，并存储为parquet格式。<!--more-->

## 技术实现

>具体实现如下：

``` scala
var midPidCard = sc.newAPIHadoopFile[LongWritable, Text, LzoTextInputFormat](args(1)).map(_._2.toString.split(",")).map(mpc => MidPidCard(mpc(1), mpc(16), mpc(0), mpc(3), mpc(18))).toDF()
```

## 注意点：

>一、使用hadoopapi指定输入文件格式为lzoText

>二、需要注意的是该方法返回的是tuple格式的rdd。

>tuple中有两个元素，

>其一是文件中的行数据偏移量，

>其二是当前行的内容，我们需要关心的是行内容，所以需要指定处理的内容为_._2。