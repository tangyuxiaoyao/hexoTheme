---
title: hbase 数据导入问题总结
date: 2017-12-26 09:43:20
tags: [hbase,bulkload,error]
---

## 需求背景

>公司现提供一批画像需要导入到hbase中，因为该画像还需要进一步处理才能作为最终的展示指标输出，所以需要自己实现业务逻辑mapreduce处理完了转换为tsv，接着转换为hfile，或者直接导入到hbase中（跳过生成hfile过程，不建议这样，比较慢）。
>本文总结的是在数据导入的过程中遇到的各种问题。
<!--more-->
## 问题解决

①: 数据量大导致的Trying to load more than 32 hfiles to one family of one region
>从字面可以看出该错误是因为在每个region的每个family中只能导入32个hfiles，超过该限制就会报错。
>该配置如下：
>/conf/hbase-site.xml
``` xml
<property>
   <name>hbase.mapreduce.bulkload.max.hfiles.perRegion.perFamily</name>
   <value>32</value>
</property>
```
>第一思路肯定是修改该配置为9999然后重启hbase搞定，但是明显在生成环境该解决方案扑街，后来查文档得知可以通过导入的时候设置参数搞定，所以准备通过该方法实现，具体做法为：当生成hfile以后我们统计下hfile的个数，当我们设计表的时候如果只用了一个列族，实际情况的确也是一个，我们的做法就是把该参数的值定位hfile的个数加一，加载该配置然后再buldload就可以解决异常。
>ps:该bug在高版本已经修复。


②: 在导入数据到hbase的过程中报错如下，然后找了很久的解决方案，终于得到解决，


``` java
LoadIncrementalHFiles loader = new LoadIncrementalHFiles(HBaseConfiguration.create(conf2));
String[] args = {dataNodesFsName+ bulkOutput, tableName};
int run = loader.run(args);
ERROR mapreduce.LoadIncrementalHFiles: Encountered unrecoverable error from region server, additional details: row '' on table 'test' at region=test,,1513078392894.d1f8ece46dc8706c9594094d6d5c15c0., hostname=hostxxx,60020,1512527608669, seqNum=2
```
>原本从字面意思可以看出是存在空的rowkey找不到对应的region（mapping不到startkey_endkey）导致的，但是过滤了多次hdfs的key,发现并没有此问题存在，最终看了很多google的解决方案试了如下的解决办法，顺利导入数据

``` shell

出现上面异常的原因是执行命令的用户权限不够： 
两种方案： 
1.给output目录（hfile的存在目录）赋予权限

hadoop fs -chmod -R 777 $output

2.切换成有权限操做的用户执行。eg:sudo -u hbase


```
> java解决方案

``` java
	public static void chmod(Configuration conf, Path path, boolean recursive) throws IOException {
		FileSystem fs = FileSystem.get(conf);
		FsPermission permission = new FsPermission((short) 00777);
		fs.setPermission(path, permission);

		if (recursive) {
			FileStatus[] fStatusArray = fs.listStatus(path);
			if (fStatusArray != null && fStatusArray.length != 0) {
				for (FileStatus fStatus : fStatusArray) {
					Path subPath = fStatus.getPath();
					if (fs.isDirectory(subPath)) {
						chmod(conf, subPath, recursive);
					} else {
						fs.setPermission(subPath, permission);
					}
				}
			}
		}
	}
```



③: 没有预分区的前提下，导入数据特别慢，只分配了两个虚拟内核和两个容器，跑的很慢，只有一个reduce在跑（HBase默认建表时有一个region)，跑的很慢，设置了reduce个数也还是没有奏效，后来查询api跟踪源码发现有如下两种解决方案:
 >1.需要生成hfile的情况，该方法调用如下:
 

``` java
	HTable table = new HTable(conf, tableName);
	job.setReducerClass(PutSortReducer.class);
	Path outputDir = new Path(hfileOutPath);
	FileOutputFormat.setOutputPath(job, outputDir);
	job.setMapOutputKeyClass(ImmutableBytesWritable.class);
	job.setMapOutputValueClass(Put.class);
	job.setOutputFormatClass(HFileOutputFormat.class);
	HFileOutputFormat.configureIncrementalLoad(job, table);

	// Use table's region boundaries for TOP split points.
	LOG.info("Looking up current regions for table " + table.getName());
	List<ImmutableBytesWritable> startKeys = getRegionStartKeys(regionLocator);
	LOG.info("Configuring " + startKeys.size() + " reduce partitions " +
	"to match current region count");
	job.setNumReduceTasks(startKeys.size());
```
>核心代码如上，可以看出该种途径reduce的的个数和regions的个数有关系，也就是你要预分区！！！
>默认地，当我们只是通过HBaseAdmin指定TableDescriptor来创建一张表时，start-end key无边界，region的size越来越大时，大到 一定的阀值，就会找到一个midKey将region一分为二，成为2个region,这个过 程称为分裂(region-split).而midKey则为这二个region的临界,但是你只是用来生成hfile的话，用的一直是一个reduce，没有裂变过程。

>2.不需要生成Hfile，直接导入情况:

``` java
	// 直接写入HBase表
	TableMapReduceUtil.initTableReducerJob(tableName, null, job);
	job.setNumReduceTasks(0);

	int regions = MetaTableAccessor.getRegionCount(conf, TableName.valueOf(table));
	if (job.getNumReduceTasks() > regions) {
	job.setNumReduceTasks(regions);
	}
```
>如上可以看出来，就算你设置了，毛用没有，还是会根据regions个数去设定reduce个数，其实也可以看出来这种方法其实也用不到reduce，但是一旦你用到的（有业务要求），你还是需要预分区！！！，这种也属于start-end key无边界，region的size越来越大时，大到 一定的阀值，就会找到一个midKey将region一分为二，成为2个region,还是需要分裂。


## 预分区方法
>合理设计rowkey 能让各个region 的并发请求 平均分配(趋于均匀) 使IO 效率达到最高.
**不预分区导致的问题**:
>1.总是往最大start-key的region写记录，之前分裂出来的region不会再被写数据，它们都处于半满状态 
>2.split是比较耗时耗资源
>执行参考语句如下:
``` sql
create 'split_table_test',{NAME =>'cf', COMPRESSION => 'SNAPPY'}, {SPLITS_FILE => 'region_split_info.txt'}  
```
>首先就是要想明白数据的key是如何分布的，然后规划一下要分成多少region，每个region的startkey和endkey是多少，然后将规划的key写到一个文件中。
>预分区需要将hbase.hregion.max.filesize设置一个较大的值，默认是10G（0.94.3 ） 也就是说单个region 默认大小是10G,如果你的其他数据源中的key设计的十分合理,可以直接用来做rowkey,可以利用该值设置大点,然后预估下总共需要预分区多少个,同时也就可以定位好endkey. 总行数(数据不重复)/(数据源文件大小/filesize)可以拿到分割数据的依据,然后利用split shell切割文件,同时tail -1 就可以拿到endkey,把endkey放到同一个splitfile中作为分区依据.


