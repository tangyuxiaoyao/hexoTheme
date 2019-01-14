---
title: ubantu16.04怎么彻底移除mysql
date: 2017-09-18 20:58:12
tags: [ubantu,mysql,移除,remove]
---


# 需求背景:
安装mysql之后不小心造成不可挽回的错误,造成崩溃,修复就没有重新安装来的省心,当然前提是在你本地损坏了.<!--more-->


## 技术实现

``` bash
sudo apt purge mysql-*
sudo rm -rf /etc/mysql/ /var/lib/mysql
sudo apt autoremove
```
