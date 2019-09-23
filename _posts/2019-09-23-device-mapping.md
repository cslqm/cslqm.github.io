---
layout: post
title: "Device mapper 机制"
subtitle: 'Device mapper'
author: "cslqm"
header-style: text
tags:
  - Linux
---

IBM确实强大，博客也写的非常好。

## LVM 的结构
LVM 的组织为三种元素：

卷（Volume）：物理 和逻辑卷 和卷组
区段（Extent）：物理 和逻辑区段
设备映射器（Device mapper）：Linux 内核模块、

### 卷
Linux LVM 组织为物理卷（PV）、卷组（VG）和逻辑卷（LV）。物理卷 是物理磁盘或物理磁盘分区（比如 /dev/hda 或 /dev/hdb1）。卷组 是物理卷的集合。卷组 可以在逻辑上划分成多个逻辑卷。

卷组是实现 n-to-m 映射的关键（也就是，将 n 个 PV 看作 m 个 LV）。在将 PV 分配给卷组之后， 就可以创建任意大小的逻辑卷（只要不超过 VG 的大小）。在图 2 的示例中，创建了一个称为 LV0 的卷组，并给其他 LV 留下了一些空间（这些空间也可以用来应付 LV0 以后的增长）。

### 区段
为了实现 n-to-m 物理到逻辑卷映射，PV 和 VG 的基本块必须具有相同的大小；这些基本块称为物理区段（PE）和逻辑区段（LE）。尽管 n 个物理卷映射到 m 个逻辑卷，但是 PE 和 LE 总是一对一映射的。

在使用 LVM2 时，对于每个 PV/LV 的最大区段数量并没有限制。默认的区段大小是 4MB，对于大多数配置不需要修改这个设置，因为区段的大小并不影响 I/O 性能。但是，区段数量太多会降低 LVM 工具的效率，所以可以使用比较大的区段，从而降低区段数量。但是注意，在一个 VG 中不能混用不同的区段大小，而且用 LVM 修改区段大小是一种不安全的操作，会破坏数据。所以建议在初始设置时选择一个区段大小，以后不再修改。

不同的区段大小意味着不同的 VG 粒度。例如，如果选择的区段大小是 4GB，那么只能以 4GB 的整数倍缩小或扩展 LV。


### 设备映射器
设备映射器（也称为 dm_mod）是一个 Linux 内核模块（也可以是内置的），最早出现在 2.6.9 内核中。它的作用是对设备进行映射 —— LVM2 必须使用这个模块。

在大多数主流发行版中，设备映射器会被默认安装，常常会在引导时或者在安装或启用 LVM2/EVMS 包时自动装载（EVMS 是一种替代 LVM 的工具，更多信息见 参考资料）。如果没有启用这个模块，那么对 dm_mod 执行 modprobe 命令，在发行版的文档中查找在引导时启用它的方法：modprobe dm_mod。

在创建 VG 和 LV 时， 可以给它们起一个有意义的名称（而不是像前面的示例那样使用 VG0、LV0 和 LV1 等名称）。设备映射器的作用就是将这些名称正确地映射到物理设备。对于前面的示例，设备映射器会在 /dev 文件系统中创建下面的设备节点：

/dev/mapper/VG0-LV0
/dev/VG0/LV0 是以上节点的链接
/dev/mapper/VG0-LV1
/dev/VG0/LV1 是以上节点的链接
（注意名称的格式标准：/dev/{vg_name}/{lv_name} -> /dev/mapper/{vg_name}{lv_name}）。

与物理磁盘相反，无法直接访问卷组（这意味着没有 /dev/mapper/VG0 这样的文件，也不能执行 dd if=/dev/VG0 of=dev/VG1）。常常使用 lvm(8) 命令访问卷组。


## Device mapper

Device mapper 分成内核部分和用户空间部分。

在内核中它通过一个一个模块化的 target driver 插件实现对 IO 请求的过滤或者重新定向等工作，当前已经实现的 target driver 插件包括软 raid、软加密、逻辑卷条带、多路径、镜像、快照等。

Device mapper 进一步体现了在 Linux 内核设计中策略和机制分离的原则，将所有与策略相关的工作放到用户空间完成，内核中主要提供完成这些策略所需要的机制。

Device mapper 用户空间相关部分主要负责配置具体的策略和控制逻辑，比如逻辑设备和哪些物理设备建立映射，怎么建立这些映射关系等等，而具体过滤和重定向 IO 请求的工作由内核中相关代码完成。因此整个 device mapper 机制由两部分组成--内核空间的 device mapper 驱动、用户空间的device mapper 库以及它提供的 dmsetup 工具。在下文中，我们分内核和用户空间两部分进行介绍。


### 内核部分

Device mapper 在内核中作为一个块设备驱动被注册的，它包含三个重要的对象概念，mapped device、映射表、target device。

- Mapped device 是一个逻辑抽象，可以理解成为内核向外提供的逻辑设备，它通过映射表描述的映射关系和 target device 建立映射。
- 从 Mapped device 到一个 target device 的映射表由一个多元组表示，该多元组由表示 mapped device 逻辑的起始地址、范围、和表示在 target device 所在物理设备的地址偏移量以及target 类型等变量组成（这些地址和偏移量都是以磁盘的扇区为单位的，即 512 个字节大小）。
- Target device 表示的是 mapped device 所映射的物理空间段，对 mapped device 所表示的逻辑设备来说，就是该逻辑设备映射到的一个物理设备。

Device mapper 中这三个对象和 target driver 插件一起构成了一个可迭代的设备树。在该树型结构中的顶层根节点是最终作为逻辑设备向外提供的 mapped device，叶子节点是 target device 所表示的底层物理设备。最小的设备树由单个 mapped device 和 target device 组成。每个 target device 都是被mapped device 独占的，只能被一个 mapped device 使用。一个 mapped device 可以映射到一个或者多个 target device 上，而一个 mapped device 又可以作为它上层 mapped device的 target device 被使用，该层次在理论上可以在 device mapper 架构下无限迭代下去。


Device mapper在内核中向外提供了一个从逻辑设备到物理设备的映射架构，只要用户在用户空间制定好映射策略，按照自己的需要编写处理具体IO请求的target driver插件，就可以很方便的实现一个类似LVM的逻辑卷管理器。Device mapper以ioctl的方式向外提供接口，用户通过用户空间的device mapper库，向device mapper的字符设备发送ioctl命令，完成向内的通信。它还通过ioctl提供向往的事件通知机制，允许target driver将IO相关的某些事件传送到用户空间。


### 用户空间部分

Device mapper在用户空间相对简单，主要包括device mapper库和dmsetup工具。Device mapper库就是对ioctl、用户空间创建删除device mapper逻辑设备所需必要操作的封装，dmsetup是一个提供给用户直接可用的创建删除device mapper设备的命令行工具。因为它们的功能和流程相对简单，在本文中对它们的细节就不介绍了，用户空间主要负责如下工作：

1. 发现每个mapped device相关的target device；
2. 根据配置信息创建映射表；
3. 将用户空间构建好的映射表传入内核，让内核构建该mapped device对应的dm_table结构；
4. 保存当前的映射信息，以便未来重新构建。

以下我们主要通过实例来说明dmsetup的使用，同时进一步说明device mapper这种映射机制。用户空间中最主要的工作就是构建并保存映射表，下面给出一些映射表的例子：

``` bash
1)
0    1024 linear /dev/sda 204
1024 512  linear /dev/sdb 766

1536 128  linear /dev/sdc 0

2) 0 2048 striped 2 64 /dev/sda 1024 /dev/sdb 0

3) 0 4711 mirror core 2 64 nosync 2 /dev/sda 2048 /dev/sdb 1024
```

例子1中将逻辑设备0~1023扇区、1024~1535扇区以及1536~1663三个地址范围分别以线形映射的方式映射到/dev/sda设备第204号扇区、/dev/sdb设备第766号扇区和/dev/sdc设备的第0号扇区开始的区域。
例子2中将逻辑设备从0号扇区开始的，长度为2048个扇区的段以条带的方式映射的到/dev/sda设备的第1024号扇区以及/dev/sdb设备的第0号扇区开始的区域。同时告诉内核这个条带类型的target driver存在2个条带设备与逻辑设备做映射，并且条带的大小是64个扇区，使得驱动可以该值来拆分跨设备的IO请求。
例子3中将逻辑设备从0号扇区开始的，长度为4711个扇区的段以镜像的方式映射到/dev/sda设备的第2048个扇区以及/dev/sdb设备的第1024号扇区开始的区域。
映射表确定后，创建、删除逻辑设备的操作就相对简单，通过dmsetup如下命令就可以完成相应的操作。
``` bash
dmsetup create 设备名 映射表文件 # 根据指定的映射表创建一个逻辑设备 

dmsetup reload 设备名 映射表文件 # 为指定设备从磁盘中读取映射文件，重新构建映射关系 

dmsetup remove 设备名          # 删除指定的逻辑设备 
```