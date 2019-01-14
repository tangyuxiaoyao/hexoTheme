---
title: spark中RDD、DataFrame和DataSet
date: 2017-12-07 15:04:53
tags: [spark,RDD,DataFrame,DataSet]
---

## 需求背景:
>分析下虽然提供了这么多的数据格式，那种适用，那种才是适合自己的。

<!--more-->

## 分析细节
### RDD
>我们在spark入门的时候接触到的是熟悉的RDD，

### DataFrame

### DataSet
>Dataset 与 RDD 相似, 然而, 并不是使用 Java 序列化或者 Kryo来序列化用于处理或者通过网络进行传输的对象. 虽然编码器和标准的序列化都负责将一个对象序列化成字节, 但编码器是动态生成的代码, 并且使用了一种允许 Spark 去执行许多像 filtering, sorting 以及 hashing 这样的操作, 而不需要将字节反序列化成对象的格式.

