---
title: 另类的数据结构HashMap
date: 2017-12-06 16:48:06
tags: [java,数据结构,HashMap]
---

## 需求背景
>分析HashMap的数据结构，put，get的过程。可以简单的理解为：链表的数组。<!--more-->

## 技术细节
>核心代码:

``` java
(k = e.key) == key || (key != null && key.equals(k)))
```


### 检索过程
>&ensp;&ensp;拿到key以后利用二次哈希函数生成hash，然后和HashTable的长度-1做与运算获得HashTable的下标，从而定位到存在于HashTable中的那个散列链表中，然后便利与key比较拿到value返回。
>ps:HashTable的定义，Entry<K,V>[] table，数组类型的Entry<K,V>

### 插入过程

>&ensp;&ensp;根据key拿到hash值(二次哈希），找到hashtable中对应的下标位置，然后判断对应的链表是否为空，如果为空，则放进去然后指针指向next，如果不为空，则根据key遍历链表判断链表中的是否存在相同key，如果存在相同的key，则返回原来的值，如果不存在相同的key，则插入到链表尾部。

### 删除过程
>&ensp;&ensp;根据Key拿到hash值（二次哈希）,然后根据拿到的hash值定位到Entry数组中的位置，然后比较散列链表中key找到对应的位置然后，改变指针的指向。

## 小知识点
>在以前大学的时候接触到的与运算拾遗。

``` java
51&9
111001
001000
=1000
```

### 总结
>&ensp;&ensp;HashMap其实也是一个线性的数组实现的,所以可以理解为其存储数据的容器就是一个线性数组。这可能让我们很不解，一个线性的数组怎么实现按键值对来存取数据呢？这里HashMap有做一些处理。
>&ensp;&ensp;首先HashMap里面实现一个静态内部类Entry，其重要的属性有 key , value, next，从属性key,value我们就能很明显的看出来Entry就是HashMap键值对实现的一个基础bean，我们上面说到HashMap的基础就是一个线性数组，这个数组就是Entry[]，Map里面的内容都保存在Entry[]里面。

``` java

    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient Entry[] table;

```
