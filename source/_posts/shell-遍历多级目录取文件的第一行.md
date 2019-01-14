---
title: shell 遍历多级目录取文件的第一行
date: 2018-04-08 19:45:10
tags: [shell,file]
---
## 需求背景

>数据取样，从当前目录下去遍历多级目录拿到每个文件的第一行数据输出到一个文件中。
<!--more-->

## 技术分析

>当听到该需求时，注意点有两点

 1. 多级目录
 2. 取文件的第一行
 
>for循环去递归遍历

## 技术实现

>viewFile.sh
``` shell
#!/bin/bash
function getdir(){
    for element in `ls $1`
    do  
        dir_or_file=$1"/"$element
        if [ -d $dir_or_file ]
        then 
            getdir $dir_or_file
        else
           head -1 $dir_or_file
        fi  
    done
}
root_dir="/home/test/multiFolder"
getdir $root_dir

```

>代码如上请笑纳。