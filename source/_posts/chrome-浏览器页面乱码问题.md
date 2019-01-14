---
title: chrome 浏览器页面乱码问题
date: 2018-05-09 09:51:20
tags: [chrome,ubantu,utf-8,tomcat]
---
## 背景描述

>在做es分词的时候用到了ik分词器要用热分词,要用到一个web容器来搭建在线词库,遇到了以下的问题.
<!--more-->

>建立词库的时候原本打算是在服务器上建立的后来,无法确认它的编码格式,后来在本地用文本建立的词典,同时也另存为了utf-8格式,然后传到了tomcat服务器的ROOT目录下,然后启动在页面预览,但是出现了乱码.


## 解决方案

>原本下意识的解决思路是修改tomcat自带的server.xml配置,但是改了以后于事无补,在页面预览还是乱码.
>后来考虑到是不是浏览器自己的编码格式问题,使用了curl直接访问这个文本静态文件,没有乱码的问题出现,从而定位去修改浏览器的默认格式问题,但是最新版本的chrome浏览器没有提供直接修改默认编码的通道,后来在chrome商店有搜到有以下这款插件可以解决问题.
>ps:(A Google Chrome extension used to modify the page default encoding for Google Chrome 55+.)
>55版本以后已经不支持在自定义字体里面修改.
[Charset][1]

  [1]: https://chrome.google.com/webstore/detail/oenllhgkiiljibhfagbfogdbchhdchml
  
  

## 写在后面
>后来发现其实跟server.xml的配置没关系,是不是新版本已经没有该问题了?(apache-tomcat-8.5.31)