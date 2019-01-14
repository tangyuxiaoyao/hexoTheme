---
title: mapreduce 温故
tags: [mapreduce,shuffer]

---


## 需求背景：
>做大数据有一段时间了，梳理下用到mapreduce的一些问题和解决方案。<!--more-->

## mapreduce
>mapreduce:顾名思义，map做映射，reduce做规约。
>主要分以下步骤：
1.输入分块
2.map
3.shuffer
4.reduce

![mapredcue流程图](/ITWO/assets/mapreduce01.jpg)
重点是shuffer阶段


## reduce个数的计算方法:

``` java
double bytes = Math.max(totalInputFileSize, bytesPerReducer);
int reducers = (int) Math.ceil(bytes / bytesPerReducer);
reducers = Math.max(1, reducers);
reducers = Math.min(maxReducers, reducers);
```
>　从计算逻辑可以看出该量由输入文件的大小以及设置的每个reduce可以处理的字节数大小决定．

![shuffer流程图](/ITWO/assets/mapreduce02.png)

## 流程细节：

###  map输出过程：

>&ensp;&ensp;&ensp;&ensp;如果没有reduce阶段，则直接输出到hdfs上，如果有reduce作业，则每个map方法的输出在写磁盘前先在内存中缓存。每个map task都有一个环状的内存缓冲区，存储着map的输出结果，默认100m，在写磁盘时，根据reduce的数量把数据划分为相应的分区(使用默认的分区算法（对输入文件的kv中对key hash后再对reduce task数量取模(reduce个数的算法见前文)),默认的hashPartioner只会作用默认分隔符分割以后的key，如果需要自定义分区，则需要你自定义二次分区比如keyfieldParttioner来实现,在每个分区中数据进行内排序，分区的个数和reduce的个数是一致的，在每次当缓冲区快满的时候由一个独立的线程将缓冲区的数据以一个溢出文件的方式存放到磁盘(这个溢写是由单独线程来完成，不影响往缓冲区写map结果的线程。溢写线程启动时不应该阻止map的结果输出，所以整个缓冲区有个溢写的比例spill.percent。这个比例默认是0.8，也就是当缓冲区的数据已经达到阈值（buffer size * spill percent = 100MB * 0.8 = 80MB），溢写线程启动，锁定这80MB的内存，执行溢写过程。Map task的输出结果还可以往剩下的20MB内存中写，互不影响。)，当整个map task结束后再对磁盘中这个map task产生的所有溢出文件做合并，被合并成已分区且已排序的输出文件。然后reduce开始fetch（拉取）map端合并好对应分区的数据，然后在reduce端合并（因为会有很多map的输出，需要合并），此时在reduce端也会进行一次sort,确保所有map的输出都排序并合并成完成以后，才会启动reduce task,所以怎么才能确保你在reduce逻辑处理时拿到的是你要的排序后的数据配合你的处理就至关重要了。

###  reducer如何知道要从哪个tasktracker取得map输出呢？

 >&ensp;&ensp;&ensp;&ensp;map任务成功完成以后，他们会通知其父tasktracker状态已更新，然后taskTracker进而通知jobTracker。这些通知在前面的心跳机制中传输。因此，对于指定作业，jobTracker知道map输出和taskTracker之间的映射关系。reducer中的一个线程定期询问jobTracher以便获取map输出的位置,直到它获得所有输出位置。
 
 
### map和reduce如何合理控制自己的个数？
 
 >&ensp;&ensp;&ensp;&ensp;map的个数是由dfs.block.size控制，该配置可以在执行程序之前由参数（见下文）控制，默认配置位于hdfs-site.xml中dfs.block.size控制，1.x的默认配置为64m,2.x的默认配置为128m,

``` java

   long goalSize = totalSize / (numSplits == 0 ? 1 : numSplits);
   long minSize = Math.max(job.getLong(org.apache.hadoop.mapreduce.lib.input.
      FileInputFormat.SPLIT_MINSIZE, 1), minSplitSize);
  long blockSize = file.getBlockSize();
  protected long computeSplitSize(long goalSize, long minSize,
                                       long blockSize) {
    return Math.max(minSize, Math.min(goalSize, blockSize));
  }
```
>&ensp;&ensp;&ensp;&ensp;从上面可以看出，最终的split size是由三个因素决定，goalsize为map输入数据除以用户自己设置的map个数（默认为1）得到的;minsize为mapred-site.xml配置的mapred.min.split.size决定，因为minSplitSize为1;第三个影响因素为blocksize,这个看配置，最终我们可以得出,如果不设置min.size,则由blocksize决定，如果设置了，则是由这两者中大的一个决定。

``` bash

set mapred.min.split.size=256000000;        -- 决定每个map处理的最大的文件大小，单位为B

方法1
set mapred.reduce.tasks=10;  -- 设置reduce的数量
方法2
set hive.exec.reducers.bytes.per.reducer=1073741824 -- 每个reduce处理的数据量,默认1GB

```
block_size : hdfs的文件块大小，默认为64M，可以通过参数dfs.block.size设置
total_size : 输入文件整体的大小
input_file_num : 输入文件的个数

（1）默认map个数
   > 如果不进行任何设置，默认的map个数是和blcok_size相关的。
   default_num = total_size / block_size;

（2）期望大小
  > 可以通过参数mapred.map.tasks来设置程序员期望的map个数，但是这个个数只有在大于default_num的时候，才会生效。
   goal_num = mapred.map.tasks;

（3）设置处理的文件大小
   > 可以通过mapred.min.split.size 设置每个task处理的文件大小，但是这个大小只有在大于block_size的时候才会生效。
   split_size = max(mapred.min.split.size, block_size);
   split_num = total_size / split_size;

（4）计算的map个数
>compute_map_num = min(split_num,  max(default_num, goal_num))

>&ensp;&ensp;&ensp;&ensp;除了这些配置以外，mapreduce还要遵循一些原则。 mapreduce的每一个map处理的数据是不能跨越文件的，也就是说min_map_num >= input_file_num。 所以，最终的map个数应该为：

``` bash
     final_map_num = max(compute_map_num, input_file_num)
```

>经过以上的分析，在设置map个数的时候，可以简单的总结为以下几点：
（1）如果想增加map个数，则设置mapred.max.split.size为一个较小的值。
（2）如果想减小map个数，则设置mapred.min.split.size 为一个较大的值。

reduce个数的设置则相对简单，要么你设置mapred.reduce.tasks的数值，要么你在hive中可以设置每个reduce可以处理的字节数，从而约束reduce的个数。

>小技巧

&ensp;&ensp;&ensp;&ensp;在hive中带空的设置参数可以打印出当前该参数的设置值。

``` sql
hive> set dfs.block.size;
dfs.block.size=268435456
hive> set mapred.map.tasks;
mapred.map.tasks=2
```


## mapreduce进度说明

### 1. Prepare

  >准备数据，抓取Map过来的输出（进度：0~33%）

 
### 2. Sort
 
  >排序阶段（进度：33%~66%）

### 3. Reduce

>真正的reduce计算阶段，执行你所写的reduce代码（进度：66%~100%）.
>如果前面66%速度很快，后面慢的话就是reduce部分没有写好；否则才是数据量大的问题。

## hadoop yarn 配置的图解

![hadoop yarn 配置的图解](/ITWO/assets/hadoop 2.0 yarn 配置项.png)

### AM的内存使用错误

``` java
Diagnostics: Container [pid=21387,containerID=container_e33_1532170420957_0001_02_000001] is running beyond physical memory limits. 
Current usage: 1.1 GB of 1 GB physical memory used; 2.7 GB of 2.1 GB virtual memory used. Killing container.
Dump of the process-tree for container_e33_1532170420957_0001_02_000001 :
        |- PID PPID PGRPID SESSID CMD_NAME USER_MODE_TIME(MILLIS) SYSTEM_TIME(MILLIS) VMEM_USAGE(BYTES) RSSMEM_USAGE(PAGES) FULL_CMD_LINE
        |- 21399 21387 21387 21387 (java) 14478 467 2856710144 281207 /usr/lib/jvm/java-8-oracle/bin/java 
		-Dlog4j.configuration=container-log4j.properties 
		-Dyarn.app.container.log.dir=/data/dev/sdb1/yarn/container-logs/application_1532170420957_0001/container_e33_1532170420957_0001_02_000001 
		-Dyarn.app.container.log.filesize=0 -Dhadoop.root.logger=INFO,CLA 
		-Dhadoop.root.logfile=syslog 
		-Djava.net.preferIPv4Stack=true -Xmx825955249 org.apache.hadoop.mapreduce.v2.app.MRAppMaster
```
>仔细预览以上的错误可以定位到以下的信息

 1. AppMaster报出的错误
 2. -Xmx825955249 设置了运行的参数值
 
>结合图示我们找下应该去集群找那些配置来定位问题

``` shell
上限参数:yarn.app.mapreduce.am.resource.mb
运行参数:yarn.app.mapreduce.am.command-opts	
```
>以上的这两参数肯定少不了，后来从集群中的确也定位到的确是是yarn.app.mapreduce.am.resource.mb该参数差的设定过小为1G，同时yarn.app.mapreduce.am.command-opts为报错中的展示信息，需要更改，因为这两项更改的都是yarn-site.xml中的配置，cdh中改完之后保存分发这些信息，然后重启集群。



| 配置文件              | 配置项                               | 设置值                                  |
| --------------------- | ------------------------------------ | --------------------------------------- |
| yarn-site.xml         | yarn.nodemanager.resource.memory-mb  | Container数量 * 每个Container的内存大小 |
| yarn-site.xml         | yarn.scheduler.minimum-allocation-mb | 每个Container的内存大小                 |
| yarn-site.xml         | yarn.scheduler.maximum-allocation-mb | Container数量 * 每个Container的内存大小 |
| mapred-site.xml       | mapreduce.map.memory.mb              | 每个Container的内存大小                 |
| mapred-site.xml       | mapreduce.reduce.memory.mb           | 2 * 每个Container的内存大小             |
| mapred-site.xml       | mapreduce.map.java.opts              | 0.8 * 每个Container的内存大小           |
| mapred-site.xml       | mapreduce.reduce.java.opts           | 0.8 * 2 * 每个Container的内存大小       |
| yarn-site.xml (check) | yarn.app.mapreduce.am.resource.mb    | 2 * 每个Container的内存大小             |
| yarn-site.xml (check) | yarn.app.mapreduce.am.command-opts   | 0.8 * 2 * 每个Container的内存大小       |

>以上为各配置的位置以及建议的设置值。

``` shell
例如：
集群的节点有 12 CPU cores, 48 GB RAM, and 12 磁盘.
预留内存= 6 GB 系统预留 + 8 GB HBase预留
最小Container内存大小 = 2 GB

如果不安装 HBase:
#Container数 = min (2*12, 1.8* 12, (48-6)/2) = min (24, 21.6, 21) = 21
每个Container的内存大小 = max (2, (48-6)/21) = max (2, 2) = 2

如果安装 Hbase：
#Container数 = min (2*12, 1.8* 12, (48-6-8)/2) = min (24, 21.6, 17) = 17
每个Container的内存大小 = max (2, (48-6-8)/17) = max (2, 2) = 2
```
