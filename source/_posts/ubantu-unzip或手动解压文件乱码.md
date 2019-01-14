---
title: ubantu unzip或手动解压文件乱码
date: 2018-10-08 11:19:02
tags: [ubantu,unzip,乱码]
---

## 使用背景
>从云盘下载的zip包中包含中文,解压出来的文件夹和文件均是乱码.
![中文乱码](/ITWO/assets/unziperror.png)

<!--more-->


----------


## 问题排查

![查看unzip的帮助命令](/ITWO/assets/ziphelp.png)

>得到 -O 可以使用 windows或者dos的编码格式去解压文件.

## 技术实现

``` shell
unzip -O GBK 笔记.zip
```
![解压的时候已经没有乱码了](/ITWO/assets/unzipsuccess.png)