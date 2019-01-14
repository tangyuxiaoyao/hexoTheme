---
title: spark 本地模式处理文件时报错
date: 2017-11-17 14:54:39
tags: [spark,local,error]
---

## 需求背景
>公司平台需要一个工具包处理，aes→sm3的工具包。写好发布到线上以后跑yarn-client模式没有错误，但是在跑取本地模式的时候会报错如下:<!--more-->

``` java
17/11/17 14:48:28 ERROR LiveListenerBus: Listener EventLoggingListener threw an exception
java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.spark.scheduler.EventLoggingListener$$anonfun$logEvent$3.apply(EventLoggingListener.scala:144)
        at org.apache.spark.scheduler.EventLoggingListener$$anonfun$logEvent$3.apply(EventLoggingListener.scala:144)
        at scala.Option.foreach(Option.scala:236)
        at org.apache.spark.scheduler.EventLoggingListener.logEvent(EventLoggingListener.scala:144)
        at org.apache.spark.scheduler.EventLoggingListener.onJobEnd(EventLoggingListener.scala:169)
        at org.apache.spark.scheduler.SparkListenerBus$class.onPostEvent(SparkListenerBus.scala:36)
        at org.apache.spark.scheduler.LiveListenerBus.onPostEvent(LiveListenerBus.scala:31)
        at org.apache.spark.scheduler.LiveListenerBus.onPostEvent(LiveListenerBus.scala:31)
        at org.apache.spark.util.ListenerBus$class.postToAll(ListenerBus.scala:53)
        at org.apache.spark.util.AsynchronousListenerBus.postToAll(AsynchronousListenerBus.scala:36)
        at org.apache.spark.util.AsynchronousListenerBus$$anon$1$$anonfun$run$1.apply$mcV$sp(AsynchronousListenerBus.scala:76)
        at org.apache.spark.util.AsynchronousListenerBus$$anon$1$$anonfun$run$1.apply(AsynchronousListenerBus.scala:61)
        at org.apache.spark.util.AsynchronousListenerBus$$anon$1$$anonfun$run$1.apply(AsynchronousListenerBus.scala:61)
        at org.apache.spark.util.Utils$.logUncaughtExceptions(Utils.scala:1617)
        at org.apache.spark.util.AsynchronousListenerBus$$anon$1.run(AsynchronousListenerBus.scala:60)
Caused by: java.io.IOException: Filesystem closed
        at org.apache.hadoop.hdfs.DFSClient.checkOpen(DFSClient.java:792)
        at org.apache.hadoop.hdfs.DFSOutputStream.flushOrSync(DFSOutputStream.java:1998)
        at org.apache.hadoop.hdfs.DFSOutputStream.hflush(DFSOutputStream.java:1959)
        at org.apache.hadoop.fs.FSDataOutputStream.hflush(FSDataOutputStream.java:130)
        ... 19 more
```

## 问题定位
>后来在网上有找到是因为sparkcontext使用完了没有关闭导致的，修改代码在主函数最后加了sc.stop()，问题解决。
