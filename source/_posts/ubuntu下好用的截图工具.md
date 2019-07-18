---
title: ubuntu下好用的截图工具
date: 2019-06-12 01:16:47
tags: [ubuntu,截图]
cover_picture: https://s2.ax1x.com/2019/07/11/ZRnztI.gif
---

## 需求背景

+ ubuntu 自带的截图工具实在是太寒颤，而且不能二次编辑和直接复制到剪贴板．

<!--more-->

## 分析

+ 那么有没有一种类似ＱＱ或者微信下的截图工具呢，后来经过多方检索和试用，得到了一款截图工具特此跟大家分享下．

## 安装与配置

### 安装

[点击这里下载安装包](http://justcode.ikeepstudying.com/wp-content/uploads/2015/12/deepin-scrot_2.0-0deepin_all.zip)



 - 下载后cd 到下载的目录　然后试用unzip 命令解压　
 - 使用sudo dpkg -i deepin-scrot_2.0-0deepin_all.deb　命令安装
> tips: 如果报错如下

``` shell
dpkg: 依赖关系问题使得 deepin-scrot 的配置工作不能继续：
 deepin-scrot 依赖于 gconf2 (>= 2.28.1-2)；然而：
  未安装软件包 gconf2。
 deepin-scrot 依赖于 python-xlib；然而：
  未安装软件包 python-xlib。

dpkg: 处理软件包 deepin-scrot (--install)时出错：
 依赖关系问题 - 仍未被配置
在处理时有错误发生：
 deepin-scrot

```
> 需要安装python-xlib．
> 解决办法:sudo apt-get install python-xlib

 - 然后试用　Deepin scrot
> 如果还有报错如下:

``` shell
Traceback (most recent call last):
  File "./deepinScrot.py", line 24, in <module>
    import gtk, os, sys, time
ImportError: No module named gtk

```
>解决方案：sudo apt-get install python-gtk2

 -  设置快捷键（直接看图）
 

![设置快捷键](/ITWO/assets/jietu_keymap_set.png)

