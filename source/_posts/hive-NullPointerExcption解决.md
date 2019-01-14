---
title: hive NullPointerExcption解决
date: 2017-12-19 19:20:24
tags: [hive,NullPointerException]
---

## 需求背景
>同事在跑去hive任务的时候，把大表放在左边，小表放在右边，出现如下错误:
> org.apache.hadoop.hive.ql.exec.mr.ExecMapper: java.lang.NullPointerException
<!--more-->
## 解决方案
>hive.auto.convert.join = false 关闭mapjoin的自动大小表切换，原因不详。