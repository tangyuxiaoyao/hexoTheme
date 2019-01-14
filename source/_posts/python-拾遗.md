---
title: python 拾遗
date: 2018-01-30 20:09:42
tags: python 
---
## 又是背景
>python 作为工作大数据mapreduce脚本支撑语言用过一段时间了,通过这篇文章查缺补漏.<!--more-->

## 展开补漏

 1. 字符串与其它类型数据连接问题

``` python
print "hello" + 666 
```
>如上代码会报错如下:

``` python
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: cannot concatenate 'str' and 'int' objects

```
解决方案如下:

``` python
>>> print "hello"+`666`
hello666
>>> print "hello"+repr(666)
hello666
>>> print "hello"+str(666)
hello666

```
>如上的解决方案,第三种比较直观,也是常用的,反引号和repr则把结果字符串转换为合法的python表达式,现在python3中已经不再使用反引号了.
>ps:str int long 属于数据基本类型转换,而repr属于函数

 2. 字符串与其它类型数据连接问题

