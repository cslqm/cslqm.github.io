---
layout: post
title: "什么是Packer"
subtitle: 'What is Packer?'
author: "cslqm"
header-style: text
tags:
  - Cloud
---


## Packer
Packer是一个自动化制作各种镜像的脚步工具，由golang编写，目前（2019-03-15）已经支持Docker，qemu和vmware的常用镜像类型，并且支持阿里云，AWS，微软的块存储接口，制作的镜像可以自动导入到云服务的存储服务上。

## Packer编译安装
官方的文档对于个操作的描述非常简单，项目目录下执行"make dev"，正在玩起来事还是挺麻烦的。当然基本和Packer项目本身无关，都是golang配置和相关工具的使用问题。（请原谅我是golang新手）

我一直以来没有写过比较大的golang轮子，习惯上都是将golang文件保存在$GOPATH/src/my\_file下，所以对于Packer也是这样来的。

	cd $GOPATH/src/my\_file
	git clone https://github.com/hashicorp/packer.git
	cd packer
	git checkout -b dev v1.3.5    #选择当前发布的tag
	make dev                      #试着编译

之后就是提示大量的包不存在的信息，并且无法获取(go get golang.org上的包)。

依赖的包太多了，完全没有兴趣从github上下载。
迷茫中，我发现Packer项目用到了govendor。这是一个非常强悍的包管理器，将依赖包全部保存到项目目录中，实现一个工程项目包含所以的依赖包。所有的依赖包都被放置到当前项目/vendor目录下。(vendor:https://github.com/kardianos/govendor)
但是Packer当前包含了所有的依赖包了，为何还是缺少包呢？这个就可能是本地需要需要安装govendor来解决编译时找包位置不对的问题。

	go get -u github.com/kardianos/govendor #安装govendor
之后你就能在$GOPATH/bin下发现一个可执行文件govendor，直接在项目目录中执行govendor就能对vendor目录下所有的包进行管理。

	govendor list #查看所有的包
	+local    (l) packages in your project
	+external (e) referenced packages in GOPATH but not in current project
	+vendor   (v) packages in the vendor folder
	+std      (s) packages in the standard library

	+excluded (x) external packages explicitly excluded from vendoring
	+unused   (u) packages in the vendor folder, but unused
	+missing  (m) referenced packages but not found

	+program  (p) package is a main package

	+outside  +external +missing
	+all      +all packages

我以为解决了所有的问题，开心的去编译了发现还是太天真。
编译时报错，找不到模块github.com/hashicorp/packer，这不就它自己嘛，它编译需要找到自己（黑人问号）。

然后我就胡乱尝试，发现一个解决方案（应该叫规避 :P ），就是将当前的$GOPATH/src/my\_file/packer目录移动到$GOPATH/src/github.com/kardianos/govendor。

	mkdir $GOPATH/src/github.com/kardianos/
	mv $GOPATH/src/my\_file/packer $GOPATH/src/github.com/kardianos/
	cd $GOPATH/src/github.com/kardianos/
	make dev

经过漫长的等待，终于在$GOPATH/bin/下看到了packer文件。




