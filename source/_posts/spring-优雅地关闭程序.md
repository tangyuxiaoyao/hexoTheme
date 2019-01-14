---
title: spring 优雅地关闭程序
date: 2018-04-27 17:11:06
tags: [spring,registerShutdownHook]
---
## 问题描述
>如何优雅地退出spring容器环境?

<!--more-->

## 细节描述

### Spring关闭钩子

>Spring在AbstractApplicationContext里维护了一个shutdownHook属性，用来关闭Spring上下文。但这个钩子不是默认生效的，需要手动调用>ApplicationContext.registerShutdownHook()来开启，在自行维护ApplicationContext（而不是托管给tomcat之类的容器时），注意尽量使用>ApplicationContext.registerShutdownHook()或者手动调用ApplicationContext.close()来关闭Spring上下文，否则应用退出时可能会残留资源。

## 补充

>Runtime.getRuntime().addShutdownHook的方法的意思就是在jvm中增加一个关闭的钩子，当jvm关闭的时候，会执行系统中已设置的所有通过addShutdownHook添加的钩子，当系统执行完这些钩子后，jvm才会关闭。
