---
title: 如何在hadoop streaming mapreduce中 使用python第三方包
date: 2018-04-11 17:26:09
tags: [hadoop,streaming,mapreduce,python,numpy]
---

## 背景描述
>当我们需要在大数据平台使用python第三方包的时候，第一印象是在集群的每台集群都安装该第三方Python包，但是这样需要集群重启，很明显此方案扑街。
<!--more-->
## 解决方案
>我们使用hadoop streaming提供你的压缩文件分发命令来使用我们的第三方Python包。


## 技术细节

>我们先了解下该命令：

``` shell
-archives	Optional	Specify comma-separated archives to be unarchived on the compute machines
```
>字面意思是上传多个以逗号分割的压缩包到集群然后会分发解压到各个计算节点。

### 打包

>打包集成python和第三方python包。

``` shell
# 需要使用第三方库如bs4,numpy等时，需要用到虚拟环境virtualenv

# virtualenv的使用
# 安装

pip install virtualenv
# 新建虚拟环境

virtualenv kxvp

# 使得虚拟环境的路径为相对路径

virtualenv --relocatable kxvp
# 激活虚拟环境

source kxvp/bin/activate
# 如果想退出，可以使用下面的命令

deactivate
# 激活后直接安装各种需要的包

# pip install XXX
# 压缩环境包

tar -czf kxvp.tar.gz kxvp

```
>上传到集群客户端


### 编写测试脚本 

>test.sh

``` shell
#!/bin/bash

hadoop fs -rmr ${2}

hadoop jar /opt/cloudera/parcels/CDH-5.4.0-1.cdh5.4.0.p0.27/jars/hadoop-streaming-2.6.0-cdh5.4.0.jar \
        -input ${1}\
        -output ${2} \
        -file map.py \
        -mapper "kx/kxvp/bin/python map.py" \
	-cacheArchive '/user/upsmart/koulb/kxvp.tar.gz#kx'\
        -jobconf mapred.reduce.tasks=0 \
        -jobconf mapred.job.name="cache_archive_test" 

```


>map.py

``` python
#!/usr/bin/env python
import numpy as np
def main(separator = ','):
	print np.__version__

if __name__ == '__main__':
    main()

```
>tip:Python的执行路径注意要加上#后面的前缀

### 结果验证

>看下集群客户端的numpy版本

``` shell
upsmart@cluster-8:~/test_tmp$ python
Python 2.7.6 (default, Oct 26 2016, 20:30:19) 
[GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import numpy
>>> print numpy.__version__
1.8.2

```


>看下mapreduce输出的结果

``` shell
upsmart@cluster-8:~/test_tmp$ hadoop fs -getmerge koulb/test/p* rs
upsmart@cluster-8:~/test_tmp$ more rs
1.14.2	
```
>不一样验证通过.

