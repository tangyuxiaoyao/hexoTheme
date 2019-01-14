---
title: mapreduce 结果集中在多个reduce输出part中的一个
date: 2018-04-27 18:19:57
tags: [mapreduce,part,reduce]
---

## 背景描述
>17G的输入数据经过map处理输出,map中只有打标记的处理,然后reduce只有cat操作,但是结果这些结果集中到一个part中

<!--more-->


## 细节分析
>先贴代码

``` java
public class HashPartitioner<K, V> extends Partitioner<K, V> {
    public HashPartitioner() {
    }

    public int getPartition(K key, V value, int numReduceTasks) {
        return (key.hashCode() & 2147483647) % numReduceTasks;
    }
}

```


>map.py
``` python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import sys
import random

def main():
	sep = "\t"
	for line in sys.stdin:
		details = line.strip().split(sep)
		card = details[0]
		print card+sep+str(random.random())





if __name__ == '__main__':
	main()

```

>start.sh

``` python
#!/bin/bash

hadoop fs -rmr ${2}
set -x
hadoop jar /opt/cloudera/parcels/CDH-5.4.0-1.cdh5.4.0.p0.27/jars/hadoop-streaming-2.6.0-mr1-cdh5.4.0.jar \
        -input ${1} \
        -output ${2} \
        -file map.py \
        -mapper "python map.py" \
        -reducer "cat" \
        -jobconf mapred.reduce.tasks=${3} \
        -jobconf mapred.job.name="reduce_test"

```



>根据默认使用的HashPartitioner可以得出是不是key分布在同一个分区中取决于reduce个数和hashcode,所以我们需要做的就是实现一个程序计算得出这些key是不是会被放到同一个分区中.

>getCode.py

``` python
# -*- coding:utf-8 -*-
import sys
def convert_n_bytes(n, b):
    bits = b*8
    return (n + 2**(bits-1)) % 2**bits - 2**(bits-1)

def convert_4_bytes(n):
    return convert_n_bytes(n, 4)

def getHashCode(s):
    h = 0
    n = len(s)
    for i, c in enumerate(s):
        h = h + ord(c)*31**(n-1-i)
    return convert_4_bytes(h)

if __name__ == '__main__':
	for line in sys.stdin:
		hashCode = getHashCode(line.strip())

		print  (hashCode & 2147483647) % reduceNum

```

>经过以上的代码验证得出这些key确实是会被放到同一个partition中,后来了解到同事用的是上一个mapreduce计算逻辑的输出中的一个part作为输入,所以也就可以捋顺之前为什么会集中到reduce输出结果中的某一个part中而不分散的缘故.


## 拓展

>由此可见这样的方法,我们可以预估到这些key是不是交给同一个reduce去处理,但是并不能准确定位到它会出现在reduce输出结果的那个part中,只会是按照排序之后依次向下排序.所以如果所有的key最终如果只会交给一个reduce处理它的结果就只会出现在part0000中.
