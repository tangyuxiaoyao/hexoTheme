---
title: 'aws s3 　递归上传和下载 '
date: 2017-11-02 20:08:32
tags: [shell,aws,s3]
---

## 需求背景
>如何在aws集群中递归下载需要的文件夹下面的内容
<!--more-->
## 技术细节
>上传

``` bash
aws s3 cp MyFolder s3://bucket-name -- recursive [--region us-west-2]
```

>下载

``` bash
aws s3 cp  s3://bucket-name  [--region us-west-2]　localfilePath -- recursive
```
