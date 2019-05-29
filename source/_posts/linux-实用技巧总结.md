---
title: linux 实用技巧总结
date: 2018-05-04 10:32:29
tags: [linux,convert,awk]
---

>总结以往在使用linux时,用到的一点小技巧或者命令集合

<!--more-->


## 批量图片格式文件转换成一个pdf

>convert *.jpg output.pdf
>ps:就是这么任性


## awk 按照某个key分发文件的妙用

### 需求背景：
>本来的需求是提取一个月的数据，但是出来以后产品又要拆分为每天的量
<!--more-->

### 技术实现
>本来打算使用python foreach去解决，但是想到以前用过awk处理过类似的问题，乍一看日期后面还有时分秒，必然又用到了substr，妙的是awk也支持，脚本如下：

``` bash
awk -F ',' '{print $3  >substr($2,1,10)".csv"}' sy*.txt;
```
>完美的解决了我的问题，第二列是时间(带有时分秒,日期格式为2017-06-13的样式)，第三列为个人标示，唯一标示一条记录，当然你也可以使用$0，完成真正意义上的拆分文件。

``` bash
awk -F ',' '{print $0 >substr($2,1,10)".csv"}'  sy*.txt;
```

## awk 批量更改某列的值
>需要更改某个文件中的某列值,用python的话很好实现,伪脚本如下

``` python
import sys

for line in sys.stdin:
    	details = line.strip().split(sep)
		details[n] = "new data"
		print sep.join(details)

```

>awk 的实现方式如下:


``` shell
awk  'BEGIN{FS=OFS=","} $2="ccc"; print $0}' aa
```
>如上的代码中需要同时制定输入和输出的分隔符,如果输出分隔符不指定的话,会导致print $0的时候默认的分隔符为空格.


## awk 实现group by

``` shell
2018-12-01,1
2018-12-01,1
2018-12-03,10
2018-11-01,11
2018-12-01,1
2018-12-01,12
2018-11-01,1
2018-12-01,1
2018-12-01,1
2018-12-02,1
2018-11-02,3
2018-12-01,2
2018-12-01,1
2018-12-04,11
2018-11-03,1
2018-12-04,1

```



``` shell
awk -F'\t' '{a[substr($1, 1, 7)] += 1}END{for (i in a){printf "%s %d\n", i, a[i]} }' file
```

+ 以上实现的是根据月份分组统计频次。