---
title: 数据集join杂谈
date: 2017-10-16 19:37:27
tags: [join,python,db,dict]
---

## 背景

> 在操作数据的时候我们经常有以下操作；<!--more-->

 1. 数据库。
 2. 各种语言中集合类。

## 技术分析
>各种集合类或者数据库中，我们操作数据无外乎，分为以下操作：增删改查。
>增：顾名思义，新的，不存在的，操作思路，新增一条记录（db），我们拿集合类中的map做类比，我们需要做的是新增一个key，并且新增一个集合类作为values,具体怎样的集合类和业务有关，所以需要做的是实例化一个集合类，然后往vaules中插入要增加的值，然后赋值。
>删：删，欲使之灭亡，必先使其疯狂，所以先找到它，捧它，让它显露出来嘚瑟，然后pass掉，delete from db.table where id=? (db)，集合类则也一样，所以判断它是否存在，然后remove掉，（顺序存储和链式存储有不同的区别）。
>改：同样的，它要首先存在，所以第一步，一样无论是where检索，或者是通过map的key去找到对应的需要修改的values值，改完了，然后重新赋值给map中的原来的key.
>查：就不说了。

## python demo 结尾:

``` python
#!/usr/bin/env python
# coding=utf-8

import sys

def read_file():
    mobile_id={}
    fileInput = open('config')
    try:
        lines = fileInput.readlines()
        cards = []
        for line in lines:
            mc = line.strip().split('\t')
            key = mc[0]
            card = mc[1]
            if(key in mobile_id.keys()):
                oldCards = mobile_id[key]
                oldCards.append(card)                
                mobile_id[key] = oldCards
            else:
                cards = []
                cards.append(card)
                mobile_id[key] = cards
        return mobile_id
    finally:
        fileInput.close()

if __name__ == '__main__':
    mobile_id = read_file()
    for data in sys.stdin:
        detail = data.strip().split("\t")
    	if(detail[2] in mobile_id.keys()):
                cards = mobile_id[detail[2]]
                for card in cards:
            	   print card+"\t"+"\t".join(detail[:2])

```
