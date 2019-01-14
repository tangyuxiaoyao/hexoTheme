---
title: ssh 免登陆设置
date: 2017-09-29 11:28:05
tags: [ssh,hadoop,spark]
---

## 需求背景:
> 在搭建大数据平台的时候，我们测试或者入门可以在local模式下进行，但是要模拟集群环境的话，就需要设置主从模式，这里已spark为例，搭建spark的standalone模式，因为涉及到远端启动所以我们需要设置ssh免登陆，<!--more-->

## 步骤如下:

 1.在各个节点上使用ssh-keygen -t rsa,过程中三次回车即可,然后在当前用户的.ssh目录下面会生成私钥文件.
 2.我们需要将这个公钥文件使用scp拷贝到其他节点并且重命名为authorized_keys．
 3.跟有交互需求的拷贝自己的公钥文件给对方从而实现该节点可以免登陆到其他节点．．
 
 拷贝公钥代码如下:
``` shell
　scp id_rsa.pub B:~/.ssh/authorized_keys
```

 