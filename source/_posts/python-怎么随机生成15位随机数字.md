---
title: python 怎么随机生成15位随机数字
date: 2017-11-02 20:13:46
tags: [python,mobile,idfa,imei]
---

## 需求背景
>测试系统的时候需要造一批手机号和idfa的设备号。<!--more-->

## 技术实现
>首先想到的是容易上手的python

设备号

``` python
"".join(random.choice("0123456789") for i in range(15))
```

手机号

``` python
random.choice(['139','188','185','136','158','151'])+"".join(random.choice("0123456789") for i in range(8))
```


