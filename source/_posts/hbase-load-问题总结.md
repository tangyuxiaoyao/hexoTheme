---
title: hbase/es load 问题总结
date: 2018-07-06 09:48:49
tags: [hbase,bulkload,es]
---

## 需求背景
>公司指标2.0开发了很多新的指标，数据量很客观，需要导入到列式存储数据库中，实现毫秒级别的响应查询，之前也有一版本的导入是导入到hbase中，但是hbase对于高并发的响应不是很理想，但凡高并发上来以后，时效性就会有影响。

<!--more-->


## 实现思路

### HBase部分

>因为数据量大的缘故，使用的是将多个HDFS文件下的数据切割然后组合成tsv，再使用PUT元素转换为HFile文件，进而使用bulkload导入hbase中。

### ES部分

>复用之前生成好的tsv文件，组装好MapWritable，然后使用EsOutputFormat完成格式化输出。


## 问题总结

### 配置文件

>这部分时间耗费在本地调试过渡到集群跑批，因为在本地集群单节点测试，你再怎么折腾也不出现配置文件找不到的问题，当然这里的前提是我使用了Properties文件做为传递配置参数的载体，在多节点测试服务器跑批的时候就会出现找不到配置文件的问题，因为该配置文件只会存在client端，而不会分发到各个其他的节点，所以会有找不到配置文件的问题出现。当然这部分也因为在使用的是ToolRunner.run(conf, new BootApplication(), args)，所以考虑过使用-files#filelink,具体用法参考字典文件分发的那篇文章，但是使用的时候，并没有达到达到预期效果，打开方式不对？
>后来的做法是先用Properties对象解析外部的绝对路径的配置问价，将拿到的配置放到一直在框架中的上下文对象Context中的Configuration中，这样只要在client端提交任务的地方，传值给它，那么无论在框架运行的什么阶段都可以读到外部配置文件中的配置项。

### HA的开启

>在本地的测试过程中，因为本地没有开启NM的HA，所以只需要配置如下内容就可以完成程序和集群的对接：

``` shell
fs.defaultFS=hdfs://hostname:8020
```
>当然如果没有配置，也可以做本地文件的测试。

>要对接开启了NM的HA的集群除了以上的配置还需要配置如下的内容:

``` shell
fs.nameservices=nameservice
#指定nameservice服务下有几个namenode
fs.ha.namenodes.nameservice=namenode33,namenode134
#指定各个namenode节点的访问链接
fs.namenode.rpc-address.nameservice.namenode33=node-01.upsmart.com:8020
fs.namenode.rpc-address.nameservice.namenode134=node-02.upsmart.com:8020
fs.client.failover.proxy.provider.nameservice=org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider

```
### habse 依赖问题

>在导入数据到hbase的过程中在一集群导入没有问题，原本以为已经可以收工，但是后来在开启的HA的另一集群测试，出现了少包的问题，这部分可能跟集群安装依赖包的寡众有关系，但是保不齐那个集群就少包了，所以你要做的是把所有的能用到的包都打到jar包里，但是打包的时候又遇到如下的问题：
>在打包的过程中有个包一直打不进去后来 

``` shell
mvn -X clean install >log
```



``` xml
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-hadoop-compat</artifactId>
            <version>1.0.0-cdh5.4.0</version>
        </dependency>

```
>定位到如下的ERROR

``` java
'dependencies.dependency.version' for commons-logging:commons-logging:jar is missing. @
```


``` xml
    <dependencies>
      <!-- General dependencies -->
      <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
      </dependency>
      <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-math</artifactId>
      </dependency>
    </dependencies>

```
>原本以为去掉这些依赖就可以万事大吉，但是并没有奏效，后来使用了最彻底的办法依照大版本相同应该改动不大的原则换了版本到1.1.10,然后打包后问题解决。
>以下方法并没有解决问题：

``` xml
            <exclusions>
                <exclusion>
                    <groupId>org.apache.hadoop</groupId>
                    <artifactId>hadoop-mapreduce-client-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>

```




>我的理解是解析pom的时候和打包去除某些依赖是不同的解析顺序


``` shell
        <dependency>
            <groupId>org.htrace</groupId>
            <artifactId>htrace-core</artifactId>
            <version>3.0.4</version>
        </dependency>

        <dependency>
            <groupId>org.apache.htrace</groupId>
            <artifactId>htrace-core</artifactId>
            <version>3.1.0-incubating</version>
        </dependency>
```

>这两个包都要加否则会报错找不到。


### mapreduce 过程报错

>beyond physical memory limits

#### map reduce  阶段报错
>解决方案：
``` shell
yarn.scheduler.minimum-allocation-mb 调节大点

```
#### 如果是只有reduce阶段报错
>解决方案：可以通过增大reduce的个数来分散reduce端的处理压力

####  reduce 100%  beyond physical memory limits

``` java

Error: org.apache.hadoop.mapreduce.task.reduce.Shuffle$ShuffleError: error in shuffle in fetcher#9
        at org.apache.hadoop.mapreduce.task.reduce.Shuffle.run(Shuffle.java:134)
        at org.apache.hadoop.mapred.ReduceTask.run(ReduceTask.java:376)
        at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:164)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1920)
        at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
Caused by: java.lang.OutOfMemoryError: Java heap space

```
>解决方案：
``` shell
-D mapreduce.reduce.shuffle.memory.limit.percent=0.1
```

### ClassNotFoundException

``` java
java.lang.ClassNotFoundException: Class org.elasticsearch.hadoop.mr.EsOutputFormat not found

```
>解决方案：

``` java
// Create job
job.setJarByClass(BootApplication.class);


```

>问题分析：看代码，一目了然。

``` java

public void setJarByClass(Class cls) {
    String jar = findContainingJar(cls);
    if (jar != null) {
        this.setJar(jar);
    }

}
```


### es导入

#### Limit of total fields [1000] in index [card_nature] has been exceeded

>解决方案：

``` shell
curl  -H "Content-Type: application/json"  -XPUT http://192.168.88.126:9200/card_nature/_settings?pretty=true -d '{"settings": 
{"index.mapping.total_fields.limit": 100000}}'

```


####  建立索引的时候报错：/usr/bin/curl: Argument list too long

>解决方案
``` shell
curl -H 'content-type: application/json' -XPUT \
  -d @- 'http://localhost:9200/card_nature' <<CURL_DATA
{
    "settings": {
                "number_of_replicas": 0,
                "number_of_shards": 6,
                "refresh_interval": "-1",
                "index.mapping.total_fields.limit": 100000
    },
        "mappings": {
            "card_nature": {
                'properties': {
                    "CP0124":{"type": "date","format": "yyyy-mm", "index": "false"},
                    "CP0125":{"type": "date","format": "yyyy-mm", "index": "false"},
                    "CP0126":{"type": "keyword", "index": "false"},
                    "CP0127":{"type": "keyword", "index": "false"},
                    "CP0128":{"type": "keyword", "index": "false"},
                }
            }
        }
}
CURL_DATA

curl -H "Content-Type: application/json"  -XPUT http://192.168.88.126:9200/card_nature/_settings?pretty=true -d '{"settings": {"refresh_interval": "5s","number_of_replicas":1}}'
curl -XPOST  http://192.168.88.126:9200/card_nature/_refresh
```

#### 建索引的时候报错

``` shell
"type": "mapper_parsing_exception",
"reason": "No handler for type [string] declared on field [CP0125]"
```
>解决方案:5.x以上已经没有string类型。如果需要分词的话使用text，不需要分词使用keyword。



### es建立索引总结
>建立索引的时候关闭索引更新，别设置副本，如果字段超过1000，您就按照上面那样设置。
>数据导入完成以后要么您开启五秒间隔更新，为了防止以后小批量的数据导入。
>要么您每次导入数据之后手动使用最下面手动更新索引。

