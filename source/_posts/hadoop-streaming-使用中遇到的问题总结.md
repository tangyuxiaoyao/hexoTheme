---
title: hadoop streaming 使用中遇到的问题总结
date: 2018-01-11 09:45:13
tags: [hadoop,streaming,python]
---

## 需求背景
>hadoop生态圈中hadoop streaming占了很大的比重,因为它很好的融合了其他脚本语言作为map,reduce来处理业务逻辑,本文重点说的是使用Python或者shell遇到的问题.
<!--more-->
## 问题汇总
### last tool output null
>报错信息如下

``` java
USER=lming_08  
HADOOP_USER=null  
last tool output: |null|  
  
java.io.IOException: Broken pipe  
```
>刚开始以为是 map 程序中对空行未处理导致的，后来发现是 map 中有对一个上传的文件进行读取，而程序对该配置文件,读取而且读取以后还会用正则去遍历模糊匹配关键词，在处理该文件时会执行 10 多亿次循环，导致超时了。
>解决方案:yield block 的使用,少用正则匹配,可以利用切词然后join来完成这样的匹配,本来mapreduce就是为了空间换时间,利用集群多个节点去并行处理逻辑.