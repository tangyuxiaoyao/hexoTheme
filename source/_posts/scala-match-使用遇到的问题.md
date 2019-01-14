---
title: scala 使用遇到的问题
date: 2018-09-13 10:45:29
tags: [scala,match]
---

## 使用背景

>在使用spark来跑统计报告任务是,有使用scala,会遇到如下的问题.

<!--more-->

+用多个开关去控制不同的使用逻辑

## 问题描述

``` scala
 val dimensionCode = args(2).toInt
 dimensionCode match {
  case AnalysisDimension.D0.getCode => {
    println("正在生成报告的维度为", AnalysisDimension.D0.getDimensions)
    //  D0(0, "行业大类,行业小类")
  }
```
+ 报错

>  如上的用法会出现以下的错误：

``` scala

  Error:(38, 33) stable identifier required, but D0.getCode found.
      case AnalysisDimension.D0.getCode => {

```
>必须要使用常量


## 解决方案

>  改动成如下的表达式就可以通过：

``` scala
    dimensionCode.toString.toInt match {
      case dimensionCode if AnalysisDimension.D0.getCode == dimensionCode => {
        println("正在生成报告的维度为:", AnalysisDimension.D0.getDimensions)
```
