---
title: java/scala 怎么给可变参数传值
date: 2018-09-07 18:06:55
tags: [java,scala,string*,string...]
---


## 业务背景

>java和scala 分别怎么给可变参数传值?

<!--more-->

## 技术实现

+ java

``` java

    public static void main(String[] args) {
		String[] args = "1,2,3".split(",", -1)
		testString(args);
    }

    static void testString(String... args) {
        for (String arg : args) {
            System.out.println(arg);
        }
    }
	
```

+ scala

``` scala

  def main(args: Array[String]): Unit = {

    val test = "1,2,3,4".split(",")

    testArgs(test: _*)

  }

  def testArgs(paths: String*) = {
    paths.foreach(println(_))
  }
  
```
