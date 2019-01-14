---
title: scrapy 入门
date: 2017-11-02 20:25:35
tags: [python,scrapy]
---

最近在自学python中的scrapy爬虫模块，以下是一些我的理解：<!--more-->

## 模块组成

![scrapy 流程图](/ITWO/assets/scrapylct.png)

## 流程建立
>自定义的spider通过请求链接访问，scheduler模块负责封装url请求的一些参数然后带着封装好的request对象去请求下载保存链接返回的资源(middlewares控制下载时候的参数:eg:设置代理),然后将Response交给spider模块中的回调函数spider处理，最终将需要的数据封装成items给item　pipelines模块去清洗。

## 编写顺序

 1. 创建一个Scrapy项目 
 2. 定义提取的Item 
 3. 编写爬取网站的 spider 并提取 Item 
 4. 编写 Item Pipeline   来存储提取到的Item(即数据)

