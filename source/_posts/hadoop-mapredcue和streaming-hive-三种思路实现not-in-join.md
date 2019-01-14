---
title: hadoop mapredcue和streaming hive 三种思路实现not in join
date: 2017-11-22 10:07:54
tags: [hadoop,mapreduce,streaming,hive,not in]
---
## 需求背景
> 公司接入市场流水，有两个路径去存储，有一天突然发现，有个路径下面流水重复，有个路径下面的流水缺失，缺失的路径下面的是生产环境用到的，所以现需要找到这部分缺失的流水，然后打补丁到该hdfs文件路径下。
<!--more-->
## 技术分析
>首先想到的技术是hive,利用传统的类sql去解决，因为以前有做过类似的需求，left outer join 去解决。
>第二想到的是hadoop streaming ，其实是领导要求的，用python脚本语言去实现缺失数据打补丁的需求。
>第三个是原汁原味的hadoop mapreduce经典案例wordcount的改良，其实也是受以前有做过wordcount和使用hadoop mapreduce实现去重联想到的，其实个人觉着这种方法是最简单的。

## 细节实现
>hive
>思路解析:在hive中一张大表和一张小表去做left ouer join ，**当小表mapping到大表中不存在的关联键时，补值null**,这样我们就可以利用小表id is  null,然后输出a.*,来实现输出大表中有而小表中没有的数据。

``` sql
select b.* from bigtable b
left outer join littletable l on b.id=l.id
where l.id is null;
```

>hadoop streaming python 
>思路解析:**streaming 中map输出并且经过shuffle以后得到的reduce输入文件是根据tab（默认的分割符，我们可以使用指定(KeyFieldBasedPartitioner)去达到将其他符号作为分隔符）分割后，同时配合加上-jobconf stream.num.map.output.key.fields=2 参数，使得前两个值作为key做排序后的数据**，所以我们可以利用该特性，初始化一个全局的变量去存储上一条数据的关联键，然后当下一条数据进来的时候去和该值比较如果一样的话，我们将该变量的值重置为空，如果不一样的话，我们输出该条数据（从这里我们可以看出，我们需要保证，两个数据源中的数据我们必须提前确保各自是已经经过去重的。）,还有一个重点是，我们需要保证给小文件中数据打上的标签必须要比大文件中的数据打上的标签值要小，才能保证优先小表中的数据作为参考key去过滤已经存在的数据。
>ps: Partitioner，默认为HashPartitioner，其根据key的hash值来决定进入哪个partition，每个partition被一个reduce task处理，所以partition的个数等于reduce task的个数


到reduce端的数据demo
``` bash
1,2,3   0
1,2,3   1
312312312       1
a,b,c   0
a,b,c   1
d,e,f   0
d,e,f   1
,.,j..  1
kljl    1
lkjl    1
ouroiu  1
zjljlj  0
```

start.sh

``` bash
#!/bin/bash

hadoop fs -test -e ${3}
if [ $? -eq 0 ]; then
hadoop fs -rmr ${3}
fi
hadoop jar /opt/cloudera/parcels/CDH-5.4.0-1.cdh5.4.0.p0.27/jars/hadoop-streaming-2.6.0-cdh5.4.0.jar \
        -D stream.num.map.output.key.fields=2 \
        -D num.key.fields.for.partition=1 \
        -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
        -input ${1} ${2}\
        -output ${3} \
        -file map.py \
        -file red.py \
        -mapper "python map.py" \
        -reducer "python red.py" \
        -jobconf mapred.reduce.tasks=5 \
        -jobconf mapred.job.name="not_in_match"
```


map.py
``` python
# -*-coding:utf-8-*-  
import sys
import md5
import os

filepath = os.environ["map_input_file"]
filename = os.path.split(filepath)[-1]

sep = '\t'
for line in sys.stdin:
	detail = line.strip()
	if filename=="littletable":
		print detail+sep+'0'
	# big table
	else:
		print detail+sep+'1'

```


reduce.py

``` python
# -*-coding:utf-8-*-  
import sys


key_exist = None
for line in sys.stdin:
	detail = line.strip().split("\t")[0]
	if(line.strip().endswith('\t0')):
		key_exist = detail
	else:
		if key_exist = detail:
			key_exist = None
		else:
			print detail

```

>hadoop mapreduce 原汤化原食
>思路解析:**在入门hadoop的时候我们都有接触过java版本的wc,其中我们可以看到redcue的输入是（k,vs）**，利用该思路，我们可以改良的下wc，来实现该功能，reduce的输出是（k,sum(vs)）,我们只要保证sum(vs)==1，然后输出该key,就可以实现输出仅在大表中才有的数据。
>彩蛋:其实利用这种特性，我们就已经可以实现去重了，因为reduce的输入已经是去重后的key然后把对应的map端输出value，拼接为集合放在后面，所以我们只要不输出reduce的value(使用NullWritable来占位)，从而实现数据去重。

map.class
``` java
    public static class Map extends MapReduceBase implements Mapper<LongWritable, Text, Text, IntWritable> {
        private Text word = new Text();
        
        public void map(LongWritable key, Text value, OutputCollector<Text, IntWritable> output, Reporter reporter) throws IOException {
            String line = value.toString();
            StringTokenizer tokenizer = new StringTokenizer(line);
            while (tokenizer.hasMoreTokens()) {
                word.set(tokenizer.nextToken());
                output.collect(word, one);
            }
        }
    }
```
reduce.class

``` java
    public static class Reduce extends MapReduceBase implements Reducer<Text, IntWritable, Text, NullWritable> {
        public void reduce(Text key, Iterator<IntWritable> values, OutputCollector<Text, NullWritable> output, Reporter reporter) throws IOException {
            int sum = 0;
            while (values.hasNext()) {
                sum += values.next().get();
            }
            if (sum == one.get()) {
                output.collect(key, NullWritable.get());
            }
        }
    }
```
>提示:在原来版本wc中我们有看到还有使用到combiner（map端的reduce）,但是使用combiner的前提是必须要保证和map的输出指定的数据类型必须是一样的，但是明显我们这里不满足，因为combinner使用的是reduce中的逻辑已经改变了输出数据的数据类型，所以要去掉或者注释掉这部分的代码。


## 写在后面
>我们处理大数据的大前提一定是要保证门清儿**各种框架的优势**，然后再在此基础上使用逻辑规约数据输出我们所需要的数据．
