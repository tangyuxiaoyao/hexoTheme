---
title: hbase 批量 提取数据
date: 2018-03-28 14:47:09
tags: [hbase,shell,python]
---

## 需求描述
>有一批卡号的需求提取全量的指标数据，这批数据存在于hbase中，所以要考虑去做mapping，来拿到数据，首先考虑的是通过hive sql语句去做join拿到要的数据，当然也是最简单，但是该hbase集群没有装hive，就是这么意外，第二策略是使用Python访问hbase(happybase),但是也要安装很多东西，最后是使用hbase shell 拿到数据以后，再批量拼装。第三种策略则是用mapreduce去读取hbase数据存储到hdfs上.
<!--more-->
## 技术分析实现

### hive

``` sql
 create external table hive_hbase_test(key int,value string)
 stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' with serdeproperties("hbase.columns.mapping"=":key,cf1:val") tblproperties("hbase.table.name"="hive_hbase_test");
```
>在现有hbase数据的基础上，我们建立一张外表打通hive和hbase数据的映射关系，然后使用我们熟悉的sqljoin然后 通过 hive -e 导出到本地收工。

+ 在操作的时候出现了以下的问题这里标注下：

``` java
org.apache.hadoop.hbase.client.RetriesExhaustedException: Can't get the locations
        at org.apache.hadoop.hbase.client.RpcRetryingCallerWithReadReplicas.getRegionLocations(RpcRetryingCallerWithReadReplicas.java:312)
```
>从表象来看是客户端去连接zk拉取region元信息的时候没有找到，所以给出的解决方案是：
>指定下zk集群

``` shell
hive -hiveconf hbase.zookeeper.quorum=slave1,slave2,master
beeline jdbc:hive2://localhost:10000 -n user -p password --hiveconf hbase.zookeeper.quorum=slave1,slave2,master
```
>最终经过验证没有问题。


### python
>此种方法没有试过，只是存在与理论中,可以参考此篇文章，优势:可以使用hbase连接池（因为是通过停thift协议通信，tcp通信）。
[Hbase实战教程之happybase](https://my.oschina.net/wolfoxliu/blog/856175)

``` python
# -*-coding:utf-8-*-
# !/usr/bin/env python
import happybase

# 初始化链接hbase的连接池
pool = happybase.ConnectionPool(size=3, host='192.168.88.57')
# 获取连接
with pool.connection() as connection:
    # 建立hbase表的操作客户端
    table = happybase.Table("xid_card", connection)
    # 查询该表的多个rowkey的所有数据
    rows_dict = dict(table.rows(['AAABAQAAABo+XKvQN57RpLZcfCr/1TC1QOPfi1nx8FSR11qmOA4xfDDqSp0=',
                                 'AAABAQAAABoK4GyYmbTXOmwNOsGnoYLIHKkGACIizeICK0M52768gSrbQKM=']))
    # 解析某个rowkey对应的值中的某个列族中的某一列值
    print rows_dict['AAABAQAAABo+XKvQN57RpLZcfCr/1TC1QOPfi1nx8FSR11qmOA4xfDDqSp0=']['n:card']

```

### hbase，shell ，python
>次篇幅的重点介绍的也是这种方法。

 1. 利用shell生成要提取的hbase shell语句
>利用shell 拼接字符的方式把所有的查询语句生成到一个文件中，当然如果需要查询的key较多你也可以多生成到几个文件中，此处不再赘述，进而使用hbase shell 语句文件得到解析到结果。
``` shell
	FILENAME=$1
	cat $FILENAME | while read LINE
	do
	echo "exists '$LINE'" >>temp_get
	echo "get 'card_md5','$LINE'" >>temp_get
	done
	hbase shell temp_get >UDcredit-20180124-1_card_res.csv

	if test $? -eq 0
	then
	exit
	fi
```
>要点:因为hbase shell查询出的结果中不会有key，所以这里我们需要暂存下key到文件中，并且肯定是要伴随着结果的输出，这样便于我们处理。所以shell逻辑如上。

 2. 利用Python处理结果文件为我们要的kv形式
 
``` python

import sys

md5_aes={}

for line in sys.stdin:
	detail = line.strip()
	if('Table' in detail):
		lines = detail.split(" ")
		md5card = lines[1]
		md5_aes[md5card]=''
	if("value" in detail):
		lines = detail.split("value=")
		sm3card = lines1
		md5_aes[md5card]=sm3card

for key in md5_aes.keys():
	print key+","+md5_aes[key]
```
>利用当前行的特殊关键字来分割当前行数据，然后利用字典，拼接kv输出得到我们要的结果。


### mapreduce 

>先贴代码
``` java
public class ReadHBaseData extends Configured implements Tool {
    static String[] cols = new String[]{};

    static String all = "all";

    @Override
    public int run(String[] args) throws Exception {

        String tableName = args[0];
        String startKey = args[1];
        String endKey = args[2];
        String outPath = args[3];
        Configuration conf = getConf();
        Job job = Job.getInstance(conf, "hbasescan");
//        conf.set("hbase.zookeeper.quorum", "mini-03.upsmart.com,mini-05.upsmart.com,mini-04.upsmart.com");
        conf.set("hbase.zookeeper.property.clientPort", "2181");
        job.setJarByClass(ReadHBaseData.class);

        Scan scan = new Scan();
        scan.setCaching(200);
        scan.setCacheBlocks(false);
        scan.setStartRow(startKey.getBytes());
        scan.setStartRow(endKey.getBytes());
//        scan.setStartRow("E26D4A603E56D297135260D34ED168C2AEBFBFCC130D2D9D55D34E36EB4CF16F".getBytes());
//        scan.setStopRow("E26E1DC81B88821716F8F47670EE95F0CB60386E4743528A43CEF9285AC71D44".getBytes());

        TableMapReduceUtil.initTableMapperJob(tableName, scan, HbaseMapper.class, ImmutableBytesWritable.class, Text.class, job);
        job.setMapOutputKeyClass(ImmutableBytesWritable.class);
        job.setMapOutputValueClass(Text.class);
        job.setReducerClass(HbaseReduce.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);
//        FileOutputFormat.setOutputPath(job, new Path("test/output"));
        FileOutputFormat.setOutputPath(job, new Path(outPath));

        //创建目标表对象
        // Execute job and return status
        return job.waitForCompletion(true) ? 0 : 1;
    }

    static class HbaseMapper extends TableMapper<ImmutableBytesWritable, Text> {


        //拿到需要的过滤的字段
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            Configuration conf = context.getConfiguration();
            cols = conf.getStrings("cols");
        }

        @Override
        protected void map(ImmutableBytesWritable key, Result value, Context context)
                throws IOException, InterruptedException {
            for (Cell cell : value.rawCells()) {
                String col = Bytes.toString(CellUtil.cloneQualifier(cell));
                if (Arrays.asList(cols).contains(all) || Arrays.asList(cols).contains(col)) {
                    context.write(key, new Text(Bytes.toString(CellUtil.cloneQualifier(cell))
                            + GlobalContant.COLON + Bytes.toString(CellUtil.cloneValue(cell))));
                }
            }
        }

    }

    static class HbaseReduce extends Reducer<ImmutableBytesWritable, Text, Text, Text> {

        @Override
        protected void reduce(ImmutableBytesWritable key, Iterable<Text> values,
                              Context context)
                throws IOException, InterruptedException {
            Text text = new Text(StringUtils.join(GlobalContant.TAB, values));
            context.write(new Text(key.get()), text);
        }


    }

    public static void main(String[] args) throws Exception {
        Configuration conf = HBaseConfiguration.create();
        Integer resCode = ToolRunner.run(conf, new ReadHBaseData(), args);
        if (resCode == GlobalContant.SUCCESS_CODE) {
            System.out.println("❀❀❀❀❀❀数据查询完成❀❀❀❀❀❀");
        } else {
            System.err.println("❀❀❀❀❀❀数据查询失败❀❀❀❀❀❀");
        }

    }

}

```

+ 核心代码

``` java
        Scan scan = new Scan();
        scan.setCaching(200);
        scan.setCacheBlocks(false);
        scan.setStartRow(startKey.getBytes());
        scan.setStartRow(endKey.getBytes());
		TableMapReduceUtil.initTableMapperJob(tableName, scan, HbaseMapper.class, ImmutableBytesWritable.class, Text.class, job);
```

+ 集成的类

``` java
HbaseMapper extends TableMapper<ImmutableBytesWritable, Text>
```
>不同于传统的mapreduce中的行字符偏移量,这里使用的是ImmutableBytesWritable,也就是hbase中的rowkey.
>不同于传统的关系型数据库,hbase中是存在多行数据属于同一条记录,这就决定了你在map中,只能是去拼装Qualifier和value.(一行的kv处理逻辑)然后在reduce中利用v3=Iterable<T>(v2),将同一个rowkey的所有的kv使用制表符作为分隔符输出到hdfs中,这样的做的好处有二:其一可以用来存储到其他集群的hbase中;其二可以将该数据存储到es中.因为他是通用的tsv结构,而且带有kv.
