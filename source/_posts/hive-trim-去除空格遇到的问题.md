---
title: hive trim() 去除空格遇到的问题
date: 2018-02-28 13:49:22
tags: [hive,trim,空格,tab,制表符,regex_replace]
---

## 需求背景

>在提取数据时，有用到使用商户号和终端号join匹配商户信息，但是返现怎么都匹配不到，以为是这两个号可能前后有空格，所以使用trim()函数处理，但还是没有匹配到结果，后来，vi文件 set list，发现 有一字段后面有^I紧跟其后，才警觉，这丫不是tab么，后来查看源码发现，hive的trim()只对空格感兴趣，所以想起可以使用replace,但是想到通用来讲，但是考虑到使用正则好点，所以又查到hive有正则替换。

<!--more-->
## 源码分析
``` java

hive 0.10.0源码

  public Text evaluate(Text s) {

    if (s == null) {

      return null;

    }

    result.set(StringUtils.strip(s.toString(), " "));

    return result;

  }
```

## 解决方案

``` sql
用法如下：

      regexp_replace(yourcolname,'\\s+','')

最后问题得以解决。​

```



