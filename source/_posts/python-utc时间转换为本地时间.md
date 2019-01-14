---
title: python utc时间转换为本地时间
date: 2018-04-27 10:13:35
tags: [python,utc,dateutil]
---

## 背景描述
> 公司业务mongo库中,存了之前很多日志,但是日志中时间没有预处理,也就是说用的是默认的utc时间,我们现实中用到的东八区的时间,公司报表系统要用到这个业务字段,导出日志的时候用的是mongoexport,后续要对时间做处理,想到了用python脚本来二次处理.

<!--more-->

## 技术实现
>因为mongo导出的时间都是字符串,所以预先要考虑的是字符串格式的数据转换为时间格式的,然后对该时间加八小时格式化输出即可.

## 技术细节:
>先贴代码为敬:

``` python
from dateutil import parser
import datetime
import sys
def main():

	for line in sys.stdin:
		details = line.strip().split(",")
		time_string = details[0]
		datetime_struct = parser.parse(time_string)
		# print type(datetime_struct)
		o = datetime.timedelta(hours=8)
		details[0] = (datetime_struct+o).strftime('%Y-%m-%d %H:%M:%S')
		print "\t".join(details)


if __name__ == '__main__':
	main()

```

>字符串时间转为时间格式用到的是dateutil里面parser,是不是很像java里面的?这个模块不是Python自带,我们需要使用pip安装名称如下

``` shell
sudo pip install python-dateutil
```
>然后使用datetime中的timedelta函数格式化八小时增量模块然后与之前的格式化的时间做相加操作,得到最终要的东八区时间.


## 拓展

>如何打印出从某个日期往后顺延指定个数的日期,枚举出来.


``` python
import datetime
start = datetime.datetime(2017,10,15)
zone = datetime.timedelta(days=1)
for i in range(184):
    day = start + zone*i
    print datetime.datetime.strftime(day,"%Y-%m-%d")
```

>思路:推算之前大概预估下你要统计多久的顺延的日子,然后使用timedelta以此累加,遍历打印就是最终想要的结果.

>ps:以此类推:某个特定小时,某个特定的月份,往后顺延,大同小异.
