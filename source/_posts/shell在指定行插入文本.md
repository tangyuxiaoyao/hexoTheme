---
title: shell在指定行插入文本
date: 2017-09-18 20:28:34
tags: [shell,sed]
---
# 需求背景
>在指定行插入特定文本<!--more-->

## 用法

``` bash
$ sed -i '第几行i文本内容' 文件
```

ps:sed -i 会直接改变文件的内容

## 特殊用法
>在a文件的第三行插入只有一个空格的空行

``` bash
sed -i '3i\ ' a
```
