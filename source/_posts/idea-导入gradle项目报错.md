---
title: idea 导入gradle项目报错
date: 2018-03-13 09:17:08
tags: [idea,gradle,android]
---

##  背景描述
>在导入一个新的gradle项目进idea时，报错如下：<!--more-->

``` java
Error:com/android/builder/model/AndroidProject : Unsupported major.minor version 52.0. Please use JDK 8 or newer.
<a href="http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html">Download JDK 8</a><br><a href="select.jdk">Select a JDK from the File System</a>
```

## 问题分析
>本来是冲着后面给的错误去解决问题（传统经验害死人），后来该改的都改了，最终还是问题依旧，这才想着从前面开始解决，问了同事，同事也说是gradle的版本问题，其实根本不是，为什么不是呢，贴出官网给的答案

``` bash
Gradle runs on all major operating systems and requires only a Java JDK or JRE version 7 or higher to be installed. To check, run java -version:
$ java -version
java version "1.8.0_121"
```
>从以上的洋文可以看出，只要是7版本以上就可以，所以就考虑到是不是idea本身的问题，在idea的设置里找了一遭，找到了**它默认开启了Android Support,后来把这项支持点掉**以后，重启idea,问题解决。