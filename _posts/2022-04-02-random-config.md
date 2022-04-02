---
layout: post
title: "systemd-random-seed启动耗时过长"
subtitle: 'systemd random seed service'
author: "cslqm"
header-style: text
tags:
  - OS
---

曾经还是老同事负责 cloud-init，当时是有个密码的加解密总是卡住很长时间。我也没有太关注。结果最近遇到非常雷同的问题：虚机启动后，很长的一段时间 ssh 登不上，但是网络是能 ping 通的。

定位问题倒是没有花费太多时间，但是了解原因的背景是真费时。

本文记录一下，免得以后又遇到，不知道原理。（原理涉及密码学，我就明白了大概，纯记录，不做做参考）


## 问题：
虚机启动后，长时间登不上去。sshd 只有一个time out信息。不能做参考。messages journal 没有明显相关的报错信息。

 
## 定位：

通过 systemd-analyze blame 确定大概时间消耗。
```
3min 3.505s  systemd-random-seed.service
50.972s      sshd.service 
```

并且可以知道 systemd-random-seed 启动非常的早，并且 sshd 依赖此服务。
```
systemctl list-dependencies systemd-random-seed.service
systemd-random-seed.service
● ├─-.mount
● └─system.slice

systemctl list-dependencies sshd.service
sshd.service
● ├─system.slice
● ├─sshd-keygen.target
● │ ├─...
● ├─sshd-keygen.target
● │ ├─...
● └─sysinit.target
● ├─...
● ├─systemd-random-seed.service
...
```

所以基本能确定：是因为 systemd-random-seed 启动耗时过长，导致 sshd 服务一直没有起来，ssh登录请求无法处理。

并且 systemd-random-seed 有个warn 日志。
```
systemd-random-seed[518]: Kernel entropy pool is not initialized yet, waiting until it is.
```

## 查到到原理知识：

linux 正常使用时需要计算很多随机数。计算随机数，需要“各种随机因子” entropy，基于这些数据来计算随机数。 正常是从 /dev/random 获得随机数。

/dev/random 计算随机数时用到 entropy pool，此池的 entropy 总数的多少直接影响所有随机数的计算快慢。所以会在开机时，尽量收集足够的 entropy。(此问题的 systemd-random-seed 运行时间过长应该就是因为 entropy pool 准备的比较慢导致的)

正常情况下默认使用的 /dev/random 是比较慢的，它是从驱动程序、环境噪音等地方收集 entropy ，然后汇总到 entropy pool。 它通常在启动时阻塞，直到收集到足够的 entropy ，然后永久解除阻塞。

/dev/urandom 它重用内部池以产生更多伪随机数（似乎是用熵池中的已有数据，计算新的熵，补充熵池）。这意味着调用不会阻塞，但输出包含的熵可能比从/dev/random的相应读取少。

还有依靠硬件的方案 HRNG。硬件实现了随机数生成（熵）。


## 解决方案：

目前试了几个网上的方案。似乎还是主要依靠rngd后台程序。内核参数 random.trust_cpu=on 似乎没大作用（内核是4.19）。

**保证镜像中有安装 rng-tools 软件包。**

rng-tools 依靠硬件的能力，如果还是累积熵池耗时过长。可以考虑扩展方案：
1.安装 haveged 软件包。（这个没试出来有明显区别）
2.修改 rngd.service，强制其使用urandom。（也没有试出来区别，但是urandom本身是非阻塞的。肯定没问题。但是arch wiki上记录的不建议使用此方案。https://wiki.archlinux.org/title/Rng-tools）

## cloud-int 老问题再分析：

cloud-init 当时设计的是非对称式加密（细节我也不懂，就知道个名字）。 其中包括 random 数据生成。因为默认情况下是使用 /dev/random 设备，但是当时熵池是不够用的，所以 /dev/random 的计算阻塞了。最终结果是 cloud-init 服务卡死，ssh也登不上去。

个人的理解：
随机数生成依靠各种足够随机的因子：熵，所有的熵汇总一起是熵池。random 设备依靠熵池计算随机数。 如果熵池中的熵不够用（大概是随机性不够），让 random 去计算随机数，它就会卡死。


仅供参考。
