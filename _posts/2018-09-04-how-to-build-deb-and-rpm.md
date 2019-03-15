---
layout: post
title: "如何在一个Makefile文件中编译生成deb包和rpm包"
subtitle: 'how to build deb and rpm package when use only Makefile'
author: "cslqm"
header-style: text
tags:
  - Makefile deb rpm
---


因为工作的原因需要将一个服务封装为使用大部分发行版的安装包，当然一个安装包肯定不行，比较有debian和redhat这两个大类别。所以至少需要有deb包和rpm包。
问题来了：如何在一台机器上，一个Makefile文件编译出deb和rpm？

## 原理

带着疑问，我开始google，但是并没找到特别好的方法。

后来我无聊在github看代码时，在一个Docker项目发现一个不错的方法。
提到Docker您应该已经知道如何做了，就是编deb包时，Docker拉Ubuntu镜像，配置好Ubuntu的环境在Ubuntu中编译，之后取到本地；编rpm包时，Docker拉CentOS镜像，配置之后在CentOS中编译，取到本地。

记录一下。
