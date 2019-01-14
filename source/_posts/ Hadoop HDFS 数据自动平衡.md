---
title: Hadoop HDFS 数据自动平衡
tags: [hadoop,hdfs,balancer]
grammar_cjkRuby: true
---

## Hadoop HDFS 数据自动平衡脚本使用方法
>在Hadoop中，包含一个start-balancer.sh脚本，通过运行这个工具，启动HDFS数据均衡服务。该工具可以做到热插拔，即无须重启计算机和 Hadoop 服务。HadoopHome/bin目录下的start−balancer.sh脚本就是该任务的启动脚本。启动命令为：‘HadoopHome/bin目录下的start−balancer.sh脚本就是该任务的启动脚本。启动命令为：‘Hadoop_home/bin/start-balancer.sh –threshold`
<!--more-->

## 影响Balancer的几个参数：



-threshold
>默认设置：10，参数取值范围：0-100
参数含义：判断集群是否平衡的阈值。理论上，该参数设置的越小，整个集群就越平衡
dfs.balance.bandwidthPerSec
默认设置：1048576（1M/S）
参数含义：Balancer运行时允许占用的带宽
示例如下：

#启动数据均衡，默认阈值为 10%
$Hadoop_home/bin/start-balancer.sh

#启动数据均衡，阈值 5%
bin/start-balancer.sh –threshold 5

#停止数据均衡
$Hadoop_home/bin/stop-balancer.sh
在hdfs-site.xml文件中可以设置数据均衡占用的网络带宽限制



``` xml
<property>
    <name>dfs.balance.bandwidthPerSec</name>
    <value>1048576</value>
    <description> Specifies the maximum bandwidth that each datanode can utilize for the balancing purpose in term of the number of bytes per second. </description>
    </property>
```
