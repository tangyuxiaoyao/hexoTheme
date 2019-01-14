---
title: python- 实现聚合（单机版和mapreduce版本）
date: 2017-10-10 15:56:56
tags: [python,date,count,sum,aggr,mean,panda]
---

## 需求背景:
>公司接口国庆期间调用量比较大，领导想要统计下今年和去年国庆期间接口每天的调用量和当天接口的调用的平均耗时．


<!--more-->


## 需求分析
>技术选型：选择了python
>同时从需求中可以了解到涉及到时间差的运算，联想到以前用过python中的datetime模块中的timedelta（时间三角洲）．

## 技术实现

### 方法一: 使用panda series解决(此解决方案由同事提供)
>代码如下：

``` python
#!/usr/bin/env python
"""
# -*- coding: utf-8 -*-
"""

import pandas as pd
import datetime
data=pd.read_csv('1610.csv')
data['requestTime']=data['requestTime'].map(lambda x:x.replace('T',' '))
data['responseTime']=data['responseTime'].map(lambda x:x.replace('T',' '))
data['requestTime']=data['requestTime'].map(lambda x:x.replace('Z',''))
data['responseTime']=data['responseTime'].map(lambda x:x.replace('Z',''))
data['requestTime']=data['requestTime'].map(lambda x:x.replace('.',' '))
data['responseTime']=data['responseTime'].map(lambda x:x.replace('.',' '))

data['delta']=pd.Series(map(lambda x,y:(datetime.datetime.strptime(x,'%Y-%m-%d %H:%M:%S %f')-datetime.datetime.strptime(y,'%Y-%m-%d %H:%M:%S %f')).total_seconds()*1000,data['responseTime'],data['requestTime']))
data['date_day']=pd.Series(map(lambda x:x[:10],data['requestTime']))
data_out=data.groupby(['uri','date_day']).delta.agg(['mean','count','sum']).reset_index()
data_out.to_csv('16.csv',index=False)


```


###方法二： 纯map解决
>这个是由两个时间的差值返回的,两个date或datetime对象相减时可以返回一个timedelta对象。
>查看timedelta源码可以找到其中返回的有以下几个方法，如下图：

![timedela中用到的方法](/ITWO/assets/timedela.png)
>可以看出来，有很多有用的方法，可以求出两个时间的天数差，秒差，微秒差．
>接口耗时我们需要的肯定是定位到毫秒即可，所以我们的解决方案是拿到微秒差然后除以1000转换为毫秒．

``` python
def compare_time(sourceTime,reqtime):
	sourceTime = datetime.strptime(sourceTime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	reqtime = datetime.strptime(reqtime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	#retval = (reqtime-sourceTime).microseconds/1000
	return retval
```
>此处这么用只算到了微秒值，但是可能存在超过一秒的调用耗时，所以我们需要使用以下的方法来实现计算毫秒差值,此方法是python 2.7以后提供的，注意使用环境。

``` python
def compare_time(sourceTime,reqtime):
	sourceTime = datetime.strptime(sourceTime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	reqtime = datetime.strptime(reqtime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	#retval = (reqtime-sourceTime).microseconds/1000
	retval = (reqtime-sourceTime).total_seconds()*1000
	return retval
```

>数据demo:/quota/usernature,2017-09-24T00:00:12.504Z,2017-09-24T00:00:12.628Z
>继续分析:既然需要分析某天某个接口唯独的指标，则我们可以将uri和日期作为key，从demo中我们可以做到使用T作为分割符分割数据一次，就可以拿到我们想要的key，然后我们使用逗号分割，计算第二项和第三项的差值（取毫秒值）做为value，同时我们考虑到同一天同一个uri的调用量很多，所以肯定是vs,我们需要定义一个list存放多个时间差值，所以中间输出值为key,vs,eg:/quota/usernature,2017-09-24,[100,200,300.....]
>然后需要做的是，遍历所有的key，当然为了方便查看，我们需要先将key排序，然后顺序查看，然后统计vs的长度，即为我们需要的该接口当天的调用量，使用list的sum函数统计总的耗时然后除以调用量就是我们需要的平均耗时。
全部代码实现如下：

``` python
# -*-coding:utf-8-*-  
import sys
from datetime import datetime


def compare_time(sourceTime,reqtime):
	sourceTime = datetime.strptime(sourceTime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	reqtime = datetime.strptime(reqtime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	retval = (reqtime-sourceTime).microseconds/1000
	return retval


def main(sep=","):
	kv = {}
	vs = []
	for line in sys.stdin:
		detail = line.strip().split(sep)
		restime = detail[1]
		reqtime = detail[2]
		value = compare_time(restime,reqtime)
		key = line.strip().split("T",1)[0].strip()
		if(key not in kv.keys()):
			vs=[]
			vs.append(value)
			kv[key] = vs
		else:
			vs.append(value)
			kv[key] = vs
	keys = sorted(kv.keys())
	for key in keys:
		cnt = len(kv[key])
		avgCnt = sum(kv[key])/cnt
		print "%s\t%d\t%f" % (key,cnt,avgCnt)


if __name__ == '__main__':
	main()

```
此方法数据量大了有点问题，统计的有问题，还在磨合中，下面给出参考如下链接写出的mapreduce两个版本的脚本。
http://www.michael-noll.com/tutorials/writing-an-hadoop-mapreduce-program-in-python/

### mapreduce版
#### 方法一:使用mapreduce的特性
>排序完使用，比较下一条和上一条的数据的key，累计调用耗时，以及次数最终的到结果。
>map.py:

``` python
# -*-coding:utf-8-*-  
import sys
from datetime import datetime


def compare_time(sourceTime,reqtime):
	sourceTime = datetime.strptime(sourceTime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	reqtime = datetime.strptime(reqtime.replace('T',' ').replace("Z",''),"%Y-%m-%d %H:%M:%S.%f")
	retval = (reqtime-sourceTime).total_seconds()*1000
	return retval


def main(sep=","):
	kv = {}
	for line in sys.stdin:
		detail = line.strip().split(sep)
		restime = detail[1]
		reqtime = detail[2]
		value = compare_time(restime,reqtime)
		key = line.strip().split("T",1)[0]
		print key+'\t'+str(value)




if __name__ == '__main__':
	main()
```

>red.py

``` python
#!/usr/bin/env python  
# -*-coding:utf-8-*-   
import sys  
  
current_key = None  
total_take = 0  
key = None
times = 0  
one = 1
for line in sys.stdin:  
    line = line.strip()  
  
    key, time_take = line.split('\t', one)  
    try:
        time_take = float(time_take)  
    except ValueError:  
        #bad line pass 
        continue  
    if current_key == key:  
        total_take += time_take
        times += one
    else:
        # if current_key ne next_key handle this logic and  print result,then init new key line data and init times
        if current_key:  
            print '%s\t%s\t%s' % (current_key,times,total_take/times)  
        total_take = time_take  
        current_key = key
        times = one  

# if stdin input over ,then handle this logic(there is no comparation after the last key compute ,so print here is necessary)
if current_key == key:
    print '%s\t%s\t%f' % (current_key,times,total_take/times) 
```
#### 方法二　使用迭代器
>同样也可以利用mapreduce中间的shuffle过程，map代码同上，reduce代码则使用yield生成迭代器同时配合groupby 工具函数来完成聚合。
>reduce.py

``` python
# -*-coding:utf-8-*-  
#!/usr/bin/env python  

from itertools import groupby  
from operator import itemgetter  
import sys  
  
def read_mapper_output(file, separator='\t'):  
    for line in file:  
        yield line.rstrip().split(separator, 1)  
  
def main(separator='\t'):  
    data = read_mapper_output(sys.stdin, separator=separator)  
    for current_word, group in groupby(data, itemgetter(0)):  
        try:
            # (float(count) for current_word,count in group)'s return is generator,if you want handle this twice,
            # you should store　generator into one collection(here:list) ,if not the generator just can be used once,
            # because the generator.next()(once used) had pointed to the end,can’t　be used more.
            res_temp = list(float(count) for current_word,count in group)
            size_count = sum(1 for _ in res_temp)
            total_count = sum(res_temp)
            print "%s%s%d%s%f" % (current_word, separator, size_count ,separator ,total_count/size_count)
        except ValueError:  
            pass  
  
if __name__ == "__main__":  
    main()  
```
>以上两种mapreduce实现逻辑，我以用蹩脚的英文解释，看不懂，就使劲儿看。

+ 知识点

>yield 的好处
Python 的老用户应该会熟悉 Python 2 中的一个特性：内建函数 range 和 xrange。其中，range 函数返回的是一个列表，而 xrange 返回的是一个迭代器。

>在 Python 3 中，range 相当于 Python 2 中的 xrange；而 Python 2 中的 range 可以用 list(range()) 来实现。

>Python 之所以要提供这样的解决方案，是因为在很多时候，我们只是需要逐个顺序访问容器内的元素。大多数时候，我们不需要「一口气获取容器内所有的元素」。比方说，顺序访问容器内的前 5 个元素，可以有两种做法：

 1. 获取容器内的所有元素，然后取出前 5 个；
 2. 从头开始，逐个迭代容器内的元素，迭代 5 个元素之后停止。

>显而易见，如果容器内的元素数量非常多（比如有 10 ** 8 个），或者容器内的元素体积非常大，那么后一种方案能节省巨大的时间、空间开销。
>现在假设，我们有一个函数，其产出（返回值）是一个列表。而若我们知道，调用者对该函数的返回值，只有逐个迭代这一种方式。那么，如果函数生产列表中的每一个元素都需要耗费非常多的时间，或者生成所有元素需要等待很长时间，则使用 yield 把函数变成一个生成器函数，每次只产生一个元素，就能节省很多开销了。特别是你在使用大的字典配置文件的时候这么用就有很大优势了,当然如果配置文件大小尚可,放到内存中去解析配置当然是最快的匹配途径.

## 总结
>mapreduce思想可以很自然的解决此类问题，但是也考虑到仅有map的情况下，怎么解决？但是这次写的代码还是存在问题，后续会解决更新。