---
title: 通过shell来实现任务状态汇报
tags: [shell,netcat]
---

## 业务背景
> 三台机器负责接受分发的数据然后运行各自的逻辑，完成之后发送信号给主节点，然后继续后续的数据分发以及跑批任务。

<!--more-->

## 技术分析

> 如果是传统的java/python程序去处理的话可以考虑到和zookeeper结合利用zk的心跳机制完成任务的定时上报和任务完成信号的汇报，shell的话怎么办？


+ 思路：在主节点配置一个监听服务器等待接受任务节点的信号，并定时轮训接受信号的文件当第一批次的任务信号接受完成，置空该信号文件并继续分发下一次批次数据给任务节点，依次类推。



## 技术实现

``` bash
#!/usr/bin/env bash


master:
nohup  nc -l -k 10000 >signalFile &

monitorDataNode(){
  susseedCnt=`grep $1 signalFile |wc -l`
  return susseedCnt
}


hive -f get_inner.sql
if [ $? != 0 ];
	then
		echo "get_inner failed"
			exit 2
else
	echo "get_inner succeed"
  hive -f stanard_inner.sh
  if [ $? != 0 ];
  	then
  		echo "stanard_inner failed"
  			exit 2
  else
  	echo "stanard_inner succeed"
    hive -f merge_inner.sql && bash  deal_out.sh
    if [ $? != 0 ];
      then
        echo "merge_inner && deal_out failed"
          exit 2
    else
      echo "merge_inner && deal_out succeed"
      monitorDataNode 0_do_get
      if [ $? == 3 ];
        then
          echo '0_do_get succeed'
          bash  1_do_reindex.sh
          .
          .
          .
      else
        monitorDataNode 0_do_get
      fi
    fi
  fi
fi



dataNode:

bash  0_do_get.sh
if [ $? != 0 ];
     then
         echo "0_do_get failed"
          exit 2
else
   echo "0_do_get succeed" |  netcat master 10000
fi


```

