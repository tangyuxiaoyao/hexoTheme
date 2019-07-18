---
title: 利用python seek 实现实时监控vpn多次尝试失败账号
date: 2019-06-06 11:15:00
tags: [python,seek,vpn]
top_img: https://s2.ax1x.com/2019/07/11/ZRnN6S.png
cover: https://s2.ax1x.com/2019/07/11/ZR1n6P.gif
---

## 需求背景

+ 运维那边有个要求，需要实时监控openvpn账号的登录状况，如果某个账号登录次数过多，需要备案提醒。

<!--more-->

## 需求分析

+ 根据需求，我们需要先分析下vpn客户端和服务器端交互的日志，分析得到哪行的日志数据带有代表性，经分析得到如下的日志具有代表性：

``` shell
AUTH-PAM: BACKGROUND: user foo failed to authenticate: Cannot make/remove an entry for the specified session
```

+ 从上面的关键日志我们可以拿到以上登录的账号名，然后把该名称记录下来然后发邮件去提醒就可以了。

## 技术实现

### shell 实现

```bash
tail -100f   /home/koulb/data/a.txt |grep "failed to authenticate"  |awk -F 'user' '{print $2}' |cut -d' ' -f2
```

### python 实现

+ shell 的实现方式并不能实现实时,tail -f 的方式管道过多,就会失去了原有的实时效能.

+ python 实现脚本如下:

``` python
# -*- encoding: utf8 -*-

import sys


# 用open打开文件
# 用seek文件指针，跳到文件最后面
# while True进行循环
# 持续不停的readline，如果能读到内容，打印出来即可



def tail_one(log_file, failed_file, failed_limit):
    failed_users = {}
    with open(log_file, "r") as r:
        r.seek(0, 2)
        while True:
            line = r.readline()
            if line:
                if "failed to authenticate" in line.strip():
                    tempDetail = line.split("user")
                    finalDetail = tempDetail[1].split(" ")
                    failed_user = finalDetail[1]

                    if (failed_user in failed_users.keys()):
                        cnt = failed_users[failed_user]
                        failed_users[failed_user] = cnt + 1
                    else:
                        failed_users[failed_user] = 1
                    for key in failed_users.keys():
                        if (failed_users[key] >= failed_limit):
                            with open(failed_file, "a+") as w:
                                w.write(key + "\n")


if __name__ == '__main__':
    if len(sys.argv) < 3:
        print "usage: python SeekFile.py 要监控的vpn日志 多次尝试登录失败的人员名单文件 失败次数"
        exit(-1)
    log_file = sys.argv[1]
    failed_file = sys.argv[2]
    failed_limit = int(sys.argv[3])
    print   "要监控的vpn日志: " + log_file
    print   "人员名单文件: " + failed_file
    tail_one(log_file, failed_file, failed_limit)

```

## 总结

+ 对于python io 的api 了解还不是很透彻,很多东西不能即拿即用,需要进一步加强.

