---
title: git commit 怎么撤销
date: 2017-09-18 20:54:49
tags: [git,commit]
---

# 需求背景:
>当你不小心提交错某个文件或者或者多提交了某些文件，并且还没有push.
<!--more-->
>可以使用如下步骤来回退到提交前的状态.	（ps:每次提交先stash,pull,stash pop 然后再commit,push）
	
``` bash
git log 
```

>会打印出所有的提交历史。
>然后定位到自己想要回退到的对应版本,找到其hash值。

``` shell
commit 0c17bf55de2054ffd6bd67714c75c0861618e3de
Author: aa
Date:   Tue Aug 8 18:14:16 2017 +0800

    bug fix: aa fix

commit 7fc618a62b219829089cb47a6c04db95157a2b4f
Author: bb
Date:   Tue Aug 8 17:56:04 2017 +0800

   bb fix bug
```
>然后使用git reset --hard commitid 回退到对应版本。