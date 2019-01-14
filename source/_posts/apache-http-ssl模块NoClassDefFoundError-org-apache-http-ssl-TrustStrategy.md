---
title: 'apache http ssl模块NoClassDefFoundError: org/apache/http/ssl/TrustStrategy'
date: 2017-09-18 20:43:52
tags: [apache,https,ssl,NoClassDefFound]
---
# 需求背景:
>项目中有链接https接口的请求所以使用了,apache的httpcomponents中的httpclient,但是启动测试的时候报错，NoClassDefFoundError: org/apache/http/ssl/TrustStrategy
<!--more-->
## 解决方案:
>然后找了下该类竟然出现在三个版本的包中，使用mvn dependency:tree >temp,找到了dubbo、disconf中也有出现，然后exclued掉.但是还是报错找不到类，后来发现该包正确使用的包在dubbo的引用下面，把该包的引用移动到上面，然后问题解决。