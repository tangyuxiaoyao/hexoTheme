---
title: 解决ubantu 谷歌浏览器打不开问题
date: 2018-03-07 14:19:10
tags: [ubantu,chrome,chromium,打不开]
---

## 背景描述

> 遇到chrome/chromium启动不了

<!--more-->

## 定位问题
>在/usr/bin/./chromium-browser启动命令，　查看log输出

>log输出如下：
>[20049:20083:0302/221800.660699:FATAL:nss_util.cc(631)] NSS_VersionCheck("3.26") failed. NSS >= 3.26 is required. Please upgrade to the latest NSS, and if you still get this error, contact your distribution maintainer.
已放弃 (核心已转储)


## 解决办法

``` bash
 sudo apt-get install libnss3 
```


