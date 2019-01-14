---
title: yarn 详解
date: 2017-12-22 16:02:53
tags: [yarn,mapreduce,hadoop]
---

## 需求背景
>yarn 作为hadoop2.0的新特性，使用了这么久的时间，把概念重新过一遍。
>温故知新，查缺补漏。<!--more-->

## 出现背景
>在hadoop1.0体系中，我们经常使用client提交任务会提交给JobTracker，然后jobTracker会根据集群现有的资源分配资源给该请求,负责任务的监控和任务的重试机制（失败→重启），TaskTracker通过心跳机制把当前任务的状态以及自己的状态报告给jobTracker。
>jobTracker的职责:

 1. 接受处理客户单提交上来的任务请求。
 2. 监控和保持TaskTracker的存活，控制map和reduce slot的数量。
 3. 监控集群上的job以及任务的执行。
 4. 指导 TaskTracker 启动 map 和 reduce 任务
 
 >TaskTracker的职责
 
1. 运行map和reduce任务。
2. 报告详情给 jobTracker。
 
>弊端:

 1. 只有一个jobTrakcer，缺乏高可用，一旦宕机，整个集群瘫痪。
 2. slot机制约束了map节点只能运行的节点类型，而且不能超过分配的slot上限，reduce亦然。

## yarn的出现

 - ResourceManager 代替集群管理器 
> 资源调度器，分配资源，监控NM，协调用户提交的应用程序，数据位置、队列、容量，ACLS(访问控制列表)。

 -  ApplicationMaster 代替一个专用且短暂的 JobTracker
 >由NM启动，协调所有任务的执行，监控任务，实现任务的重试机制;AM和tasks都运行在NM控制的container容器中运行。
 >一个节点上的容器数量：由配置的参数和除去后台进程和操作系统之外的节点资源总量（比如总 CPU 数和总内存）共同决定。
 >得益于应用程序的代码都转移到了AM中，所以各种分布式框架只要实现了AM,定制个性化的服务，都会受YARN的支持。（MapReduce、Giraph、Storm、Spark、Tez/Impala、MPI ）

 - NodeManager 代替 TaskTracker 


 - 一个分布式应用程序代替一个 MapReduce 作业
 
 