---
title: ssh 免密码登录以及多种用途共存
date: 2018-10-12 16:51:58
tags: [scp,ssh,git]
---

## 使用背景
>在日常使用ssh时,我们有以下几种使用场景

 1. ssh 远程登录远端的机器
 2. scp 与其他的主机 上传或者下载数据
 3. ssh 生成邮箱的公钥用于免密拉取git的项目

<!--more-->

## 实现和原理分析

``` shell
	（1）在HOSTA机器上产生公钥和私钥
		   ssh-keygen -t rsa
		   
	（2）需要将HOSTA机器的公钥复制给HOSTB机器
		  ssh-copy-id -i .ssh/id_rsa.pub root@HOSTB
```

+ 图解

![详细交互过程](/ITWO/assets/ssh-rsa.png)

## 写在后面
>既然ssh生成的公钥可以用来这么多用途,那么怎样让用于免密码登录/拷贝的公钥和用来拉取git项目的公钥共存呢?


``` shell
# man ssh-add
NAME
     ssh-add — adds private key identities to the authentication agent
     ssh-add adds private key identities to the authentication agent, ssh-agent(1).  When run
     without arguments, it adds the files ~/.ssh/id_rsa, ~/.ssh/id_dsa, ~/.ssh/id_ecdsa,
     ~/.ssh/id_ed25519 and ~/.ssh/identity.  After loading a private key, ssh-add will try to load
     corresponding certificate information from the filename obtained by appending -cert.pub to
     the name of the private key file.  Alternative file names can be given on the command line.

# 从以上的解释可以看出通过该命令可以解决此问题
ssh-add ~/.ssh/*_rsa
```
+ ssh-add操作
![ssh-add 操作过程](/ITWO/assets/ssh-add.png)


+ 过程测试
![ssh-add 操作过程](/ITWO/assets/ssh-git.png)

>ps: 用作一种用途时,生成id_rsa和id_rsa.pub之后记得重命名,然后使用ssh-add添加该密钥.
