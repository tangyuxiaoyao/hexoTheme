---
title: 给hive mapreduce分配队列和优先级
date: 2018-01-10 16:46:12
tags: [hive,mapreduce,queue,priority]
---

## 需求背景
>集群资源有限,考虑到有多个部门要用到,所以使用不同部门使用不同队列的方式来使用集群资源,这里我们使用的是Capacity Schedule分配策略,顾名思义,是给各个队列分配相应占比的资源来独立使用,而队列内部则还是使用yarn默认的FIFO Schedule分配策略,这样可以保证每个队列可设定一定比例的资源最低保证和使用上限，同时，每个用户也可以设定一定的资源使用上限以防止资源滥用。而当一个队列的资源有剩余时，可暂时将剩余资源共享给其他队列。<!--more-->

## 使用详解
>hive --hiveconf mapreduce.job.queuename=root.fast --hiveconf mapreduce.job.name='job name' -f  etl.hql
>hive
>set mapreduce.job.queuename=root.fast;
>set mapreduce.job.name='job name'
>
>yarn application  -movetoqueue  application_1478676388082_963529  -queue  root.etl #迁移任务到另一个队列

ps:
>作业提交到的队列：mapreduce.job.queuename

>作业优先级：mapreduce.job.priority，优先级默认有5个:LOW VERY_LOW NORMAL（默认） HIGH VERY_HIGH