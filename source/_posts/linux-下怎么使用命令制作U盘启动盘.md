---
title: linux 下怎么使用命令制作U盘启动盘
date: 2018-10-24 13:27:07
tags:
---

## 使用背景

>ubantu自带的工具没有响应,后来查到可以使用命令的方式来格式化u盘和制作装机盘.

<!--more-->


## 技术实现

``` shell

#unmout the mount pan
sudo umount /dev/sdb

#format u pan
sudo mkfs.vfat /dev/sdb –I


#make u pan to a pan which can play as a bootpan
sudo dd if=~/iso/deepin-15.7-amd64.iso of=/dev/sdb

```

