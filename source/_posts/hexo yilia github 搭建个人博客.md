---
title: hexo yilla 和github 结合搭建个人博客 
tags: [hexo,yilla,github]
toc: true
comments: true
---


# 为什么考虑这样的搭配方式?
## 构建需求
 >现阶段有很多的技术网站都带有给想要展示自己的一些技术入门以及技术研究的平台,也就常见的技术博客.
  但是大多都不满足于个性化定制,偶然间接触到markdown,当然简书等这样的平台也支持markdown,但是出于个人的独占情节,还是更倾向于搭建一个独立的自己可控的blog.
<!--more-->
## 技术实现(快速搭建)
>考虑到如果从零开始,买空间,选域名,构建主体框架,渲染静态页面,一套走下来,未免本末倒置,博客注重的应该是文章的可读性以及质量,当然ui需要一定的可观瞻性.幸运的是遇到了hexo,给人一种转角遇到爱的小确幸.
	  ps:Hexo is a fast, simple & powerful blog framework powered by Node.js.
	  从官网的解释可以看出,我们需要安装Node.js,当然要和gitbub结合,你需要申请一个github账号,申请账号的步骤,此处就不再累述.
	  笔者使用的系统是ubantu 16.04
	  
>传送门:[github账号申请]( https://github.com/join)

### 现在演示安装Node.js.
> 在 Github 上获取 Node.js 源码：

``` bash
$ sudo git clone https://github.com/nodejs/node.git
```

 1.修改目录权限：
``` bash
$ sudo chmod -R 755 node
$ cd node
$ node -v
```
 2.使用 ./configure 创建编译文件
``` bash
$ sudo ./configure 
```
 3.这一步，可能时间有点长，耐心等待
``` bash
$ sudo make 
```
 4.最后
``` bash
$ sudo make 
```

>install 查看版本

``` bash
$ node -v
```
>v0.10.25 如果node不是最新的，node有一个模块叫n，是专门用来管理node.js的版本的。使用npm（NPM是随同nodejs一起安装的包管理工具）安装n模块

``` bash
$ sudo npm install -g n  
```
>然后，升级node.js到最新稳定版

``` bash
$ sudo n stable  
```
>旧版本的 npm，也可以很容易地通过 npm 命令来升级，命令如下：

``` bash
$ sudo npm install npm -g
```

### 安装hexo

>执行以下的命令:

``` bash
$ npm install hexo-cli -g
$ hexo init blog 此处会新建一个新的目录存储hexo的一些初始化的文件.
$ cd blog
$ npm install
$ hexo server 此处会生成一个新的本地预览 访问http://localhost:4000 就可以访问本地的默认主题.
```

### 新建github仓库
>新建一个仓库,然后选择public权限(写博客不就是为了别人看,进而监督自己进步么,所以public),指定git分支,使用默认的master分支即可,因为这个仓库就你一个看门的,这里面也是你的.然后记住自己的clone地址.
>ps:记得保存.

### 将gitbub仓库和hexo主题绑定
>编辑_config.yml:
	
``` less
  type: git
  repo: https://github.com/tangyuxiaoyao/ITWO.git
  branch: master
```
>repo配置的地址为上文已经提及过的项目仓库clone地址.
>而且这里有看到仓库后面带有子资源路径所以参考配置文件中的注释,需要将root对应的配置改为仓库的名称资源路径.

``` vim
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
root: /ITWO/
```


### 发布主题到github仓库

``` bash
npm install hexo-deployer-git --save
hexo clean && hexo g && hexo d
```
 1. 安装hexo发布模块deploy
 2. 清除缓存
 3. 生成静态页面
 4. 发布到github(每次改完以后也是这么稳妥发布)
### 生成新的文章

``` bash
$ cd source/_posts/
$ hexo new "shell在指定行插入文本"
```


>然后就会生成一个为该名称的md文件,根据md语法编辑内容,完成以后发布即可.
ps:可以用hexo clean && hexo g && hexo s在本地生成预览效果,避免发布到github上的效果不尽人意.


![古人笑比庭中树,一日秋风一日疏](/ITWO/assets/jscy.jpg)