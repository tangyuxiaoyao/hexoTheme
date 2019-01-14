---
title: hadoop streaming 配置文件和字典文件分发方式
date: 2018-05-10 20:48:07
tags: [hadoop streaming,python,cacheFile,file,files]
---

## 背景描述

>通常情况下我们在hadoop streaming中通过input来设置我们需要处理的数据,然后逐条遍历打标记,从而在reduce端可以区别对待做聚合或者join操作.但是我们需要用到一些配置文件怎么办?

<!--more-->

## 技术细节

>这里使用python来实现,我们的第一思路是:

``` shell
python test.py dict.txt
-file dict.txt
```
>当然这种方案只可以针对本地文件,一方面这样的文件每次都要分发带来很大的局限性(如果文件很大怎么办?是不是会很耗费性能?显然是的),另一方面如果我们需要使用的文件在hdfs上怎么实施?
>先贴下官网的普及文档

### Hadoop Streaming中的大文件和档案
``` shell

任务使用-cacheFile和-cacheArchive选项在集群中分发文件和档案，选项的参数是用户已上传至HDFS的文件或档案的URI。这些文件和档案在不同的作业间缓存。用户可以通过fs.default.name.config配置参数的值得到文件所在的host和fs_port。

这个是使用-cacheFile选项的例子：

-cacheFile hdfs://host:fs_port/user/testfile.txt#testlink
在上面的例子里，url中#后面的部分是建立在任务当前工作目录下的符号链接的名字。这里的任务的当前工作目录下有一个“testlink”符号链接，它指向testfile.txt文件在本地的拷贝。如果有多个文件，选项可以写成：

-cacheFile hdfs://host:fs_port/user/testfile1.txt#testlink1 -cacheFile hdfs://host:fs_port/user/testfile2.txt#testlink2
-cacheArchive选项用于把jar文件拷贝到任务当前工作目录并自动把jar文件解压缩。例如：

-cacheArchive hdfs://host:fs_port/user/testfile.jar#testlink3
在上面的例子中，testlink3是当前工作目录下的符号链接，它指向testfile.jar解压后的目录。

下面是使用-cacheArchive选项的另一个例子。其中，input.txt文件有两行内容，分别是两个文件的名字：testlink/cache.txt和testlink/cache2.txt。“testlink”是指向档案目录（jar文件解压后的目录）的符号链接，这个目录下有“cache.txt”和“cache2.txt”两个文件。

$HADOOP_HOME/bin/hadoop  jar $HADOOP_HOME/hadoop-streaming.jar \
                  -input "/user/me/samples/cachefile/input.txt"  \
                  -mapper "xargs cat"  \
                  -reducer "cat"  \
                  -output "/user/me/samples/cachefile/out" \  
                  -cacheArchive 'hdfs://hadoop-nn1.example.com/user/me/samples/cachefile/cachedir.jar#testlink' \  
                  -jobconf mapred.map.tasks=1 \
                  -jobconf mapred.reduce.tasks=1 \ 
                  -jobconf mapred.job.name="Experiment"

$ ls test_jar/
cache.txt  cache2.txt

$ jar cvf cachedir.jar -C test_jar/ .
added manifest
adding: cache.txt(in = 30) (out= 29)(deflated 3%)
adding: cache2.txt(in = 37) (out= 35)(deflated 5%)

$ hadoop dfs -put cachedir.jar samples/cachefile

$ hadoop dfs -cat /user/me/samples/cachefile/input.txt
testlink/cache.txt
testlink/cache2.txt

$ cat test_jar/cache.txt 
This is just the cache string

$ cat test_jar/cache2.txt 
This is just the second cache string

$ hadoop dfs -ls /user/me/samples/cachefile/out      
Found 1 items
/user/me/samples/cachefile/out/part-00000 

$ hadoop dfs -cat /user/me/samples/cachefile/out/part-00000
This is just the cache string   
This is just the second cache string

```
>以上的demo可以看出可以帮助我们通过这种途径去预览分发后的hdfs配置文件.

### 实践校验

``` shell
#!/bin/bash

hadoop fs -rmr ${2}

hadoop jar /opt/cloudera/parcels/CDH-5.4.0-1.cdh5.4.0.p0.27/jars/hadoop-streaming-2.6.0-mr1-cdh5.4.0.jar \
        -files testSymlink.py,hdfs:///user/upsmart/koulb/dict.txt#test \
		# -file testSymlink.py \
		# -cacheFile ,hdfs:///user/upsmart/koulb/dict.txt#test \
        -input ${1} \
        -output ${2} \
        -mapper "cat" \
        -reducer "python testSymlink.py" \
        -jobconf mapred.reduce.tasks=1 \
        -jobconf mapred.job.name="testSymlink"
```
>tips:此处的files等价于file + cacheFile,其实官网有很长一段时间的版本已经在推崇这样的做法,但是还是在兼容这样的模式用法,file用来上传分发本地的文件到集群,cacheFile顾名思义用的是集群上已经现存的资源.
> -file option is deprecated, please use generic option -files instead.
``` python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

# 打开一个文件
# 通过软链接去访问配置文件
fo = open("test")
print "文件名: ", fo.name

# 遍历文件打印每行的数据
for line in fo:
        print line.strip()

# 关闭打开的文件
fo.close()
```

## 细节描述


### map使用细节

``` shell
        -mapper "python testSymlink.py" \
        -jobconf mapred.reduce.tasks=0 \
```

>看细节:

``` shell
JobSubmitter: number of splits:3

#  配置文件结果输出
文件名:  test
a
b
c
d
e
文件名:  test
a
b
c
d
e
文件名:  test
a
b
c
d
e
```
>从上可以看出我们使用的配置文件打印了三次,和我么你的splits个数一致.也就是说和map个数一致.

### recude使用细节

``` shell
        -reducer "python testSymlink.py" \
        -jobconf mapred.reduce.tasks=1 \
```

>看细节

``` shell
# 配置文件输出
文件名:  test
a
b
c
d
e
```

>同样可以得出输出的次数和reduce个数完全一致


### 使用总结

>综上可得,完全达到了配置文件分发的目的,可以保证每个map和reduce都可以使用到所需的配置文件.

>tips:**JobSubmitter: number of splits:3**,
>**3**  = **files:2** + **input1**
