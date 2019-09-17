---
layout: post
title: "Windows 操作系统盘符分配失败"
subtitle: 'Do not support more than 27 drive letters'
author: "cslqm"
header-style: text
tags:
  - OS
---


最近遇到云盘类型磁盘在windows的服务管理器中，无法自动分配盘符的问题。所以关注windows文件系统比较多。


众所周知，windows支持盘符，这是从DOS时代留下来的，两个软盘使用掉A和B，之后支持的硬盘从D盘开始。所以盘符只有A-Z这些个了，当然微软想加CC、DD的话肯定也能支持。

以下内容节选自云风的blog，https://blog.codingnow.com/2017/02/windows_path_sep.html 。

> Windows 的前身是微软的 Dos 系统。我见过的最古老的 MS Dos 是在我同学家的一台旧 IBM PC/XT 上。10M 的硬盘、只有 256K 内存，配置的系统是 PC(MS) Dos 2.0 。
> 
> 知道这个 2.0 版的 Dos 比它的前身最大的改进是什么吗？它比 MS DOS 1.0 多支持了硬盘，以及层级目录结构。
> 
> 同时期的主流电脑是 Apple II ，它的原生操作系统（Apple Dos）是不支持硬盘的，软盘上只有一个根目录。MS Dos 1.0 也一样。
> 
> 除了 IBM PC，Apple II ，那个年代个人电脑品牌其实非常多。比如我的第一台电脑就是港产的 Z80 机器（laser 310 ）。当时 8 位机上最流行的个人操作系统是一个叫做 CP/M 的系统。不过当时流行直接用汇编写程序，这个 CP/M 必须跑在 Z80(8080) 的指令集上。当年 Apple II 上流行过一种叫做 Z80 卡的扩展件，就是为了可以跑 CP/M 系统。
> 
> 微软起家为 IBM 的 8086 系列 PC 写操作系统时，就借鉴了 CP/M 的一些东西，其中就有盘符这个东西。在没有硬盘，及内存小于软盘容量的年代，配置两个软盘驱动器是最方便的，所以就有了 A B 两个盘符（方便数据对拷）。每个文件都可以写成 A:FILENAME.EXT 的形式。这就是盘符的由来。
> 
> btw, 早年 IBM 想和 CP/M 合作没谈成，后来 PC 流行后，CP/M 又反回来兼容了 MS-DOS 跑在 PC 系统上，改名字叫 DR-DOS ，应该很多同学有印象。
> 
> MS DOS 发展到 2.0 时，由于 IBM 给 PC/XT 增加了 10M 的硬盘，所以盘符就被扩展到了 C: 表示硬盘。储存空间的增加导致了必须增加文件目录结构。可为啥微软选择了反斜杠，而不是 Unix 系列中已经很广泛的 / 呢？
> 
> 这是因为，MS Dos 已有的很多命令行工具。当时微软做开发的人有 DEC 的背景，DEC 的操作系统上是用 / 做命令行参数分割符，而不是 Unix 系列用的 - ，就这样沿用到了 MS Dos 里。btw， 我读大学时第一次接触 Linux ，感觉输入最不习惯的就是用 - 而不是 / 。
> 
> 为了防止混淆，目录风格符就不能再使用 / 了，DEC 系统中用的是点 "." ，可 MS Dos 学了 CP/M 用 . 做文件后缀名分割，所以就用了让后代程序员深恶痛绝的反斜杠 \ 。


盘符分配失败是什么原因呢？
在windows中，对于新盘的使用，需要先将其初始化为gpt或mbr，之后新建卷，这样会分配出一个volume，然后给volume分配盘符，个人感觉分配盘符的过程和 Linux中的 mount 过程非常相似。而且 windows ntfs 确实支持挂载。比如 diskpart assign、mountvol。

牛人画的 他自己理解的 windows 文件系统树结构。 https://blog.csdn.net/dog250/article/details/100140882

![windows tree filesystem](/img/in-post/my-windows-os-filesystem.png)

所以到这里时猜想原因应该是两个 1. 盘符申请失败 2. "挂载失败"。 但是 windows event 中又没有任何错误日志。 :-) 值得一提的是，非云盘类型硬盘可以正常分配，所以应该是第二个原因的可能性比较大。

您怎么看？
