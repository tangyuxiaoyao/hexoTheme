---
title: spark 项目启动的时候报出如下错误
date: 2017-09-19 14:01:46
tags: [spark,not,match]
---
## 需求背景:
>在spark打包启动的时候报错信息如下:

``` java
“class "javax.servlet.FilterRegistration"'s signer information does not match signer information of other classes in the same package”
```
## 细节分析:

>从字面意思可以看出是因为pom中有多个jar包中出现了该类，在程序中或者依赖的程序去使用该类的时候导致无法唯一确定jar包的来源.

## 解决方案:

>对于此类问题处理的方式是：
找到该类所在的父gav然后在其中exesusion掉该jar包，因为exesusion只可以指定到ga,所以就会把该类所有的jar包去掉，但是如果这么用又会出现该类找不到（no class found）,此时的解决办法就是引入一个确定的jar包版本指定gav,放在exclusion父类之前.
