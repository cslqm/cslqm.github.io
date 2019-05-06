---
layout: post
title: "Incomplete"
subtitle: 'arp incomplete'
author: "cslqm"
header-style: text
tags:
  - Linux
---


我在写一个自动生成可用ip地址的脚步，大致包括生成ip，通过ping的方式检查是否被占用，通过检查本地（本地是网关）arp表查看是否有被用。

因为是先ping，后检查arp。所以就要了这样的脚步

``` bash
ping -w 5 ip_addr
arp -an | grep   ip_addr  | grep -v grep
```

但是会发现即使ip地址从未使用过（ping不通，且arp表没有记录）arp总是已零返回。
手动执行arp发现还真有返回结果，类似于下边这个：
"? (192.168.123.123) at <incomplete> on packerbr0"

可是我执行脚步前，明明没有这些信息啊？谁加上的。

仔细检查发现，原来是ping先执行导致的。举个例子：
在一个控制台输入"ping -w 5 192.168.122.133"，让其运行着。（要保证这个ip不在arp表缓存中，且ping不通）
另一个控制台输入"arp -an | grep 192.168.122.133"，就会打印出关于这个ip的arp信息。"? (192.168.122.133) at <incomplete> on XXXX"

什么原因呢， 目前在网上看到的说法是 三层网络的ping icmp进行的同时会建立ip地址到mac地址的信息，但是对于这个ip icmp是得不到包含mac地址回复的，所以只能先已incomplete表示mac地址。

