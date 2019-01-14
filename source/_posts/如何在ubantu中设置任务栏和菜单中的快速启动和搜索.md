---
title: 如何在ubantu中设置任务栏和菜单中的快速启动和搜索
date: 2017-09-19 14:19:49
tags: [ubantu,shell,dash]
---
## 背景需求:
>一些应用程序（例如很多.sh程序）如果想在Ubuntu中添加到Dash home中进行快速的启动.<!--more-->

## 场景分析:
>需要找到/usr/share/applications这个目录，其中存放的全部是dash中的启动器，将你需要的程序xxx添加其中即可。

## 技术实现:
>具体操作步骤为：

``` bash
cd /usr/share/applications sudo gedit xxx.desktop
```

>打开需要编辑的文本内容为：

``` ini
[Desktop Entry] 
Version=1.0 
Name=robomongo
GenericName=robomongo
Keywords=mongo;robo;robomongo (此配置是在程序搜索时，可以使用罗列的关键字)
Exec=/home/username/xxx.sh（这个是启动程序需要执行的文件路径名） 
Terminal=false 
Icon=/home/username/xxx.png（这个是图标，这个一般程序里面不带，可以去官网找logo） 
Type=Application 
Categories=Development
```

>这样就可以在dash中看到xxx的启动器图标了，也可以直接将其添加锁定到launcher