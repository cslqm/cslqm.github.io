---
layout: post
title: "如何将远端仓库的特定tag同步到本地，并创建分支"
subtitle: 'get tag of origin, create branch from the tag'
author: "cslqm"
header-style: text
tags:
  - Linux
---

如果基于已有开源项目做二次开发，就需要将个人项目中维护一个上游分支。社区的代码更新了，个人的项目也同步更新。

开源代码发布新版本，都会新建tag。可以将新tag获取到本地项目，并用此tag创建本地upsteam分支，这样就成功获取开源项目的最新release的代码。

``` bash
# 获取远端的名为tagname的tag到本地，名字不变保存为本地tag 
git fetch origin tag tagname
# 从某一个名为tagname的tag创建一个新分支 new-branch-name
git branch new-branch-name tagname
```

完成上边的操作后，就获取到最新的发布代码。之后可merge到自己维护的项目上，非常方便。