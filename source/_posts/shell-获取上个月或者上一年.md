---
title: shell 获取上个月或者上一年
date: 2017-09-21 16:26:33
tags: [shell,date]
toc: true
comments: true
---
## 需求背景:
>开发过程中会遇到很多情况，有用到date函数去获得上个月（上一年）的需求，比如用来获取上个月的日志或者上个月的数据之类的诉求。<!--more-->

## 解决方案:

``` bash
lastMonth=`date -d "1 month ago" +%Y_%m`
lastYear=`date -d "1 year ago" +%Y_%m`
```

## 结言：
>感觉很直观，很容易记住。
