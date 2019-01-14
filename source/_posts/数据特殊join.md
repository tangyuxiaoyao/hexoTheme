---
title: 数据特殊join
date: 2019-01-03 20:05:06
tags: [shell,comm,python]
---

## 需求背景
> 有两份计费数据一份是测试账号的，一份是正式账号，现计费策略如下：

+ 测试账号调用过的key如果在正式账号调用记录中出现:
 1. 测试账号调用该key的次数大于等于正式账号，则计费次数为测试出现的次数减去正式账号出现该key的次数。
 2. 测试账号调用该key的次数如果小于正式账号，则计费次数为正式账号调用的次数。
+ 测试账号和正式账号的没有重复的则分别计费即可。

<!--more-->

## 实现思路

+ 相同部分不好计费，需要特殊计费通过Python来实现。
+ 不同部分通过awk追加到一个文件中，然后使用sort|uniq -c 来统计词频即可。（此处不能使用comm，使用该命令一定要先排序去重，那计费数据不就丢了？）



## 技术实现

``` shell

step1:compute the key cnt with tag

sort Tcard > s_tcard
sort Fcard > s_fcard

cat s_tcard|uniq -c |awk -F' ' '{print $2"\tT_"$1}' >tag_tcard
cat s_fcard|uniq -c |awk -F' ' '{print $2"\tF_"$1}' >tag_Fcard


step2: handle the common part

sort tag* |python handleCnt.py >common.tsv


step3:handle the diff part


awk  'NR==FNR{a[$0]}NR>FNR{ if(!($1 in a)) print $0}' s_tcard s_fcard >diff_temp
awk  'NR==FNR{a[$0]}NR>FNR{ if(!($1 in a)) print $0}' s_fcard s_tcard >> diff_temp


sort diff_temp |uniq -c |awk -F' ' '{print $2"\t"$1}' >diff.tsv


step4:cat comm and diff to final.tsv

cat common.tsv diff.tsv >final.tsv




```


``` python
#!/usr/bin/env python
# vim: set fileencoding=utf-8
import sys

def main(separator = '\t'):
    initDict = {}
    card = None
    for data in sys.stdin:
        detail = data.strip().split(separator)
        if(detail[1].startswith("F")):
            card = detail[0]
            cnt = detail[1].split("_")[1]
            initDict[card] = cnt
        elif(detail[0] in  initDict.keys() and detail[1].startswith("T_")):
            cnt =  detail[1].split("_",1)[1]
            if(initDict[card]>=cnt):
                print card+separator+initDict[card]
            else:
                print card+separator+str(int(cnt)-int(initDict[card]))


if __name__ == '__main__':
    main()

```
