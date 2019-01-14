---
title: shell 合并hdfs文件 分发 并加文件校验和
date: 2018-03-06 10:40:13
tags: [shell,hadoop,hdfs,awk,spark,scala,file,suffix]
---

## 背景描述

> 提供一批带有城市标识的商户号，然后在流水中匹配到卡号，最终通过设备号和卡号的映射关系，输出每个城市商户商户对应的设备号。<!--more-->


## 具体实现

``` scala
object MapDevice {

  case class MidCard(card: String, mid: String)

  case class CardDevice(card: String, device: String)


  case class CityMid(city: String, mid: String)

  val comma = ","
  val tab = "\t"

  // 从业务逻辑来看直接判断分割以后的列数更能够过滤掉脏数据，当然空行也被pass了。
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("商户数据抓取设备号").setMaster("local[*]")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)


    // .filter(_.matches("\\S+")) 过滤空行
    // 考虑到只有city和mid的前提下，就会产生重复数据，所以要过滤以后再建表。
    import sqlContext.implicits._
    val midFilePath = args(0)
    val cardMids = sc.textFile(midFilePath)
      .filter(_.trim.split(comma, -1).length == 6)
      .map(line => {
        val details = line.split(comma, -1)
        details(0) + comma + details(4)
      }).distinct().map(line => {
      val details = line.split(comma, -1)
      CityMid(details(0), details(1))
    }).toDF()

    cardMids.registerTempTable("cityMid")

    //    cardMids.show()

    val transFilePath = args(1)

    val todayMCDF = sc.newAPIHadoopFile[LongWritable, Text, LzoTextInputFormat](transFilePath)
      .filter(_._2.toString.split(comma, -1).length == 47)
      .map(_._2.toString.split(comma, -1))
      .map(mpc => MidCard(mpc(0), mpc(1)))
      .toDF()

    todayMCDF.registerTempTable("midcard")


    val cardDevicePath = args(2)


    // .filter(_.trim.length > 0) 过滤空行
    // _.device.matches("\\S+") 过滤匹配到的设备号为空
    val cardDeviceDF = sc.textFile(cardDevicePath)
      .filter(_.trim.split(tab, -1).length == 10)
      .map(line => {
        val items = line.split(tab, -1)
        val card = items(0)
        val deviceId = items(3)
        CardDevice(card, deviceId)
      }).filter(_.device.matches("\\S+")).toDF()


    cardDeviceDF.registerTempTable("carddevice")

    //    cardDeviceDF.show(10)

    val resDF = sqlContext.sql("select city,device from (select card,city from cityMid c  join cardmid m on c.mid=m.mid ) c join carddevice d on d.card=c.card group by city,device")


    //    resDF.show(5)
    val resSavePath = args(3)


    resDF.map(value => value.mkString(comma)).saveAsTextFile(resSavePath)

  }
```



``` bash

#!/usr/bin/env bash
set -x

if [ $# -ne 4 ]; then
        echo "参数个数为$#个"
        echo "请按照如下格式输入参数"
        echo "bash map-device.sh 商户信息配置路径 hardin流水路径 卡号和设备号对应关系路径 结果文件存放路径"
        exit
fi



# koulb/cardmid/onemid koulb/cardmid/testtrans.lzo* koulb/cardmid/card_device_sm3 koulb/cardmid/out

midconfPath=$1
transPath=$2
cardDevicePath=$3
resPath=$4



spark-submit \
--class com.upsmart.MapDevice  \
--num-executors 100 \
--executor-memory 6G \
--executor-cores 4 \
--driver-memory 1G \
--conf spark.default.parallelism=1000 \
--conf spark.storage.memoryFraction=0.5 \
--conf spark.shuffle.memoryFraction=0.3 \
--master yarn-client map-device-1.0.jar $midconfPath $transPath $cardDevicePath $resPath


if [ $? -ne 0 ];then
   exit
fi

fileName=`date -d "1 hour ago"  +"%Y%m%d%H"`

fileFullName=$fileName".csv"
hadoop fs -getmerge $resPath $fileFullName

if [ $? -ne 0 ];then
   exit
fi


awk -F ',' '{print $2  >$1"_""'$fileName'"".csv"}' $fileFullName

rm $fileFullName

for i in `ls`;
do
	fileSuffix=`echo ${i##*_}`
    if [ $fileSuffix == $fileFullName ];then
    	fileSum=`md5sum $i  |cut -d' ' -f1`
    	echo $fileSum >$i".md5"
    fi
done


if test -e .*.crc
then
    rm -f .*.crc
fi



```


## 问题总结


### 怎么才算是有效的数据过滤？
> 原本的思路是过滤掉空行就行，但是后续也会有角标越界的风险存在，从而过滤的最终条件为当前行按照分隔符切割以后，必须是标准列数
> ps: java和scala都存在这样的bug,如果多行数据的最后一列为空，直接使用spit,得到的列数会少1,当然你去便利该数组，也会少个元素，必须这样使用spilt(str,-1),
> 核心代码：

``` scala?linenums
_._2.toString.split(comma, -1).length == 47

```


### 如果匹配到的设备号为空怎么办？
> 开发之初，没有考虑到该因素，后来有问到数据提供的同事，不排除会存在设备好为空或者空格占位，所以使用正则过滤掉这些无意义的输出
> 代码:filter(_.device.matches("\\S+"))

### 怎么多列的配置文件的前提下，过滤掉同个城市下会存在重复的mid
>不多说，直接贴代码

``` scala
   // 考虑到只有city和mid的前提下，就会产生重复数据，所以要过滤以后再建表。
    import sqlContext.implicits._
    val midFilePath = args(0)
    val cardMids = sc.textFile(midFilePath)
      .filter(_.trim.split(comma, -1).length == 6)
      .map(line => {
        val details = line.split(comma, -1)
        details(0) + comma + details(4)
      }).distinct().map(line => {
      val details = line.split(comma, -1)
      CityMid(details(0), details(1))
    }).toDF()
```

### 如何在最终结果文件中按文件分发？（shell实现以及理论上的scala实现）


#### shell

``` bash

fileName=`date -d "1 hour ago"  +"%Y%m%d%H"`
fileFullName=$fileName".csv"
hadoop fs -getmerge $resPath $fileFullName
awk -F ',' '{print $2  >$1"_""'$fileName'"".csv"}' $fileFullName

```
>难点1：按第一列去分发第二列内容，参考本博客的其他文章中的awk分发文件。
>难点2：在awk中引用变量的值

``` bash


koulb@koulb-ubantu:~$ a="test"
koulb@koulb-ubantu:~$ awk 'BEGIN{print "'$a'"}' 
test

```

#### scala
> 最终的文件为多个城市对应设备号的结果，需要做的是，把第一列group by 去重，然后遍历该城市列表拿拿到对应的设备号，生成对应的结果文件。