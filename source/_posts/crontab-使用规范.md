---
title: crontab 使用规范
date: 2018-01-02 10:17:01
tags: [crontab,log]
---
## 背景描述
>在使用crontab定制定时任务时,没有结果输出,想去查看下log.但是没有找到,通过一下途径来定位问题.
<!--more-->
## 问题分析:
>①该服务有没有开启?

``` shell
查看该服务是否开启?
sudo /etc/init.d/crond status 
如果没有开启,启动该服务.
sudo /etc/init.d/crond start
```
>②确认该任务开启以后,查看log定位问题,然后并没有看到log.

``` shell?linenums
修改rsyslog
sudo vim /etc/rsyslog.d/50-default.conf
cron.*              /var/log/cron.log #将cron前面的注释符去掉 
重启rsyslog
sudo  service rsyslog  restart
查看crontab日志
less  /var/log/cron.log 

```
## 用法核心
>语法规范:

``` shell
 m h  dom mon dow   command
 分 时 每月的第几日 每年的第几个月 每周的第几天 可执行任务
```
> 使用demo

``` shell
# 每天的10:05输出"1232"到log中
05 10 * * * echo '1232' >> /home/testuser/test.log 2>&1
```


