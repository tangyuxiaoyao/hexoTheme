---
title: spring boot 和layui 集合实现简单的提供页面加解密工具
date: 2017-12-28 13:56:53
tags: [spring,boot,layui]
---

## 需求背景:
>公司现阶段有很多的加密方式,过段时间会最终只有一个SM3,所以过渡阶段会有很多的解密加密的查询解析诉求,所以借用一个下午的时间想实现下提供一个页面供同事使用,来完成以下功能:

 1. 明文到现公司的各种加密(自定义脱敏,aes,sm3).
 2. 脱敏或者aes的数据的解密操作.
 3. 脱敏和aes数据到sm3的操作.
<!--more-->
##  技术分析:
>ui 选择的时候选择了快速上手的layui,可以快速封装各种DOM,java框架选择了快速搭建的spring boot,然后一步到位实现功能.

## 技术实现

### 整合核心点
>springboot静态资源相对路径问题,我们都知道我们在与前端绑定数据交互的时候会在resource目录下，
建立一个static目录用来存放用到的css和js,并建立一个template目录用来存放h5页面，此时我们需要注意的是
**这两个目录对于springboot项目来讲是透明和屏蔽掉的**，所以我们在h5页面设计js路径和css路径的时候要考虑到该影响因素.

### h5和layui
>具体实例如下:

``` bash
.
├── java
├── resources
│   ├── banner.txt
│   ├── faviconbak.ico
│   ├── favicon.ico
│   ├── static
│   │   ├── js
│   │   └── layui
│   │       ├── css
│   │       │   ├── layui.css
│   │       ├── layui.all.js
│   │       └── layui.js
│   └── templates
│       └── index.html

```
>以下为在index.html中的正确写法:

``` html
<link rel="stylesheet" href="layui/css/layui.css" media="all"/>
```

>错误写法(自以为的写法):

``` html
<link rel="stylesheet" href="../static/layui/css/layui.css" media="all"/>
```
>layui和后台ajax交互部分

``` javascript
<script>

    layui.use(['form'],function(){
        var form = layui.form
        ,$ = layui.$
        ,layer = layui.layer;


          //监听提交
        form.on('submit(demo1)', function(data){
            $.ajax({
                url: "/data/handle",
                contentType: "application/json",
                data: data.field, //请求的附加参数，用json对象
                method: 'GET',
                success: function (res) {
                    layer.msg(res, {
                        time: 20000, //20s后自动关闭
                        btn: ['明白了', '知道了', '哦']
                    });
                }
            })
            return false;
        });


    });



</script>
```


### java部分:

``` java
package cn.upsmart.controller;

import cn.upsmart.constant.GlobalContant;
import cn.upsmart.utils.HandleUtils;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * Copyright (C), 2017, 公司
 *
 * @author koulb
 * @version 0.0.1
 * @desc 处理数据核心类
 * @date 17-12-26
 */
@RestController
public class DataHandleController {
    
    
    public static final String aesKey = "fdadssfafdadssfa";
    
    @RequestMapping("/data/handle")
    public String dataHandle(@RequestParam String inputdata,
                             @RequestParam String dataSource,
                             @RequestParam String dataType,
                             @RequestParam String handleType) {
        
        switch (dataType) {
            case GlobalContant.REAL:
                switch (handleType) {
                    case GlobalContant.SMARTENCODE:
                        return HandleUtils.realToTm(inputdata, dataSource);
                    case GlobalContant.SM3:
                        return HandleUtils.realToSM3(inputdata, dataSource);
                    case GlobalContant.AES:
                        return HandleUtils.realToAes(inputdata, dataSource, aesKey);
                    default:
                        return "加密真实卡号时,发现错误的加密类型";
                }
            case GlobalContant.SMARTENCODE:
                switch (handleType) {
                    case GlobalContant.AES:
                        return HandleUtils.tmToAES(inputdata, dataSource);
                    case GlobalContant.SM3:
                        return HandleUtils.tmToSM3(inputdata, dataSource);
                    case GlobalContant.SMARTDECODE:
                        return HandleUtils.tmToReal(inputdata, dataSource);
                    default:
                        return "处理反脱敏数据时,发现错误的处理方式";
                }
            case GlobalContant.AES:
                String tmdata;
                switch (handleType) {
                    case GlobalContant.DAES:
                        tmdata = HandleUtils.daes(inputdata, aesKey);
                        return HandleUtils.tmToReal(tmdata, dataSource);
                    case GlobalContant.SM3:
                        tmdata = HandleUtils.daes(inputdata, aesKey);
                        String realdata = HandleUtils.tmToReal(tmdata, dataSource);
                        return HandleUtils.realToSM3(realdata, dataSource);
                    case GlobalContant.SMARTENCODE:
                        return HandleUtils.daes(inputdata, aesKey);
                    default:
                        return "处理AES加密的数据时,发现错误的处理方式";
                }
        }
        
        
        return "未知错误请联系管理员";
    }
    
    
}

```

