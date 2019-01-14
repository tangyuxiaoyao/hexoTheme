---
title: 安装es遇到的问题和解决方案
date: 2018-05-28 09:03:14
tags:  [es,]
---

## 需求背景

>在安装es过程中遇到的问题总结。

<!--more-->


## 问题一

>1. max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]


``` shell
sudo vi /etc/sysctl.conf
在最后一行加入如下配置
vm.max_map_count=262144

```

----------


## 问题二

``` shell
bootstrap.memory_lock: true
```

>开起如上配置启动报错如下

``` java
2. bootstrap checks failed  memory locking requested for elasticsearch process but memory is not locked

These can be adjusted by modifying /etc/security/limits.conf, for example:
#allow user 'upsmart' mlockall
upsmart soft memlock unlimited
upsmart hard memlock unlimited
If you are logged in interactively, you will have to re-login for the new limits to take effect.
These can be adjusted by modifying /etc/security/limits.conf, for example:
#allow user 'upsmart' mlockall
upsmart soft memlock unlimited
upsmart hard memlock unlimited
If you are logged in interactively, you will have to re-login for the new limits to take effect.

```

>根据以上提示要修改以下内容

``` shell

sudo vim /etc/security/limits.conf
#allow user 'upsmart' mlockall
upsmart soft memlock unlimited
upsmart hard memlock unlimited

```


----------

## 写在后面
>改的过程中发现启动程序不起作用，后来发现提示下面还有一句话提示的是如果不起作用，需要重新建立登录，后来关闭该会话，然后重启程序。（细节很重要！！！）
