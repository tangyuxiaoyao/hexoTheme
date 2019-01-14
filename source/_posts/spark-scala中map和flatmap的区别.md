---
title: spark scala中map和flatmap的区别
date: 2017-09-19 17:20:15
tags: [spark,scala,map,flatmap]
toc: true
comments: true
---
# 需求背景：

> 统计相邻两个单词出现的次数。<!--more-->

## 技术实现

``` scala
val s="A;B;C;D;B;D;C;B;D;A;E;D;C;A;B"

s: String = A;B;C;D;B;D;C;B;D;A;E;D;C;A;B

 val data=sc.parallelize(Seq(s))

data.collect()

res0: Array[String] = Array(A;B;C;D;B;D;C;B;D;A;E;D;C;A;B)       
```
> 截止目前位置是一个String类型的数组。

``` scala
val mapTemp=data.map(_.split(";"))

scala> mapTemp.collect

res4: Array[Array[String]] = Array(Array(A, B, C, D, B, D, C, B, D, A, E, D, C, A, B))
```
> map操作在于处理之前和处理之后的数据类型是一致的。

``` scala
val mapRs=data.map(_.split(";")).map(x=>{for(i<-0 until x.length-1) yield (x(i)+","+x(i+1),1)})

mapRs.collect

res1: Array[scala.collection.immutable.IndexedSeq[(String, Int)]] = Array(Vector((A,B,1), (B,C,1), (C,D,1), (D,B,1), (B,D,1), (D,C,1), (C,B,1), (B,D,1), (D,A,1), (A,E,1), (E,D,1), (D,C,1), (C,A,1), (A,B,1)))
```
> *map输出的数据类型:Array[scala.collection.immutable.IndexedSeq[(String, Int)]]*
> 而flatMap会把一类集合类的数据抹平从而展示的效果是元素的方式，真实效果:从Vector中遍历然后罗列出来。

``` scala
val flatMapRs=data.map(_.split(";")).flatMap(x=>{for(i<-0 until x.length-1) yield　(x(i)+","+x(i+1),1)})

flatMapRs.collect

res3: Array[(String, Int)] = Array((A,B,1), (B,C,1), (C,D,1), (D,B,1), (B,D,1), (D,C,1), (C,B,1), (B,D,1), (D,A,1), (A,E,1), (E,D,1), (D,C,1), (C,A,1), (A,B,1))
```
> flatmap输出的数据类型:Array[(String, Int)]*

## 最终结果

``` scala
val flatMap= data.map(_.split(";")).flatMap(x=>{for(i<-0 until x.length-1) yield (x(i)+","+x(i+1),1)}).reduceByKey(_+_).foreach(println)
```
(A,E,1)

(C,D,1)

(B,D,2)

(D,B,1)

(C,A,1)

(C,B,1)

(E,D,1)

(D,A,1)

(B,C,1)

(D,C,2)

(A,B,2)

## 如果只是做抹平操作该怎么做
>Array(Array(A, B, C, D, B, D, C, B, D, A, E, D, C, A, B)) => Array(A, B, C, D, B, D, C, B, D, A, E, D, C, A, B)
>=> for (i <- Array) print i 
>  data.map(_.split(";")).flatMap(item => item).foreach(println)
A
B
C
D
B
D
C
B
D
A
E
D
C
A
B


## 源码走一趟

``` scala
  def flatMap[B](f: A => GenTraversableOnce[B]): Iterator[B] = new AbstractIterator[B] {
    private var cur: Iterator[B] = empty
    def hasNext: Boolean =
      cur.hasNext || self.hasNext && { cur = f(self.next).toIterator; hasNext }
    def next(): B = (if (hasNext) cur else empty).next()
  }
```


## 总结一下:
> 处理以前一定要意识到你自己最终拿到的数据的格式,如果是集合类的你则最终需要需要flat(抹平操作),从而才能使用一些函数对每个元素进行处理.


## reduceByKey算数因子解释：

> Basically reduceByKey function works only for RDDs which contains key and value pairs kind of   elements(i.e RDDs having tuple or Map as a data element). It is a transformation operation   which means it is lazily evaluated.We need to pass one associative function as a parameter,   which will be applied to the source RDD and will create anew RDD as with resulting values(i.e.  key value pair). This operation is a wide operation as data shuffling may happen across the   partitions.【本质上来讲，reduceByKey函数（说算子也可以）只作用于包含key-value的RDDS上，它是  transformation类型的算子，这也就意味着它是懒加载的（就是说不调用Action的方法，是不会去计算的  ）,在使用时，我们需要传递一个相关的函数（_+_）作为参数，这个函数将会被应用到源RDD上并且创建一个新的  RDD作为返回结果，这个算子作为data Shuffling 在分区的时候被广泛使用】  

## flatMap
>Spark RDD flatMap function returns a new RDD by first applying a function to all elements of this RDD, and then flattening the results.
>Important points to note are,

 - flatMap is a transformation operation in Spark hence it is lazily evaluated
 -  It is a narrow operation as  it is not shuffling data from one partition to multiple partitions 
 - Output of flatMap is flatten
 - flatMap parameter function should return array, list or sequence (anysubtype of scala.TraversableOnce)