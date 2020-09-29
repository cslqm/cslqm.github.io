---
layout: post
title: "刷机流程粗略记录"
subtitle: 'My MI'
author: "cslqm"
header-style: text
tags:
  - 生活
---

手机太老了，还在用红米 note4x，总是卡，手痒痒的不行，刷了个机。
短时间换手机可能性不大，估计以后还会用到，记录一下。


## 前期准备

先将卡刷包拷入到手机的内部存储或者 SD 卡。因为刷入 twrp 后，需要立即进行刷入卡刷包的操作，这样可以避免第三方的 REC 被小米官方 REC 覆盖。


## 手机解锁

手机开启开发者模式。

先下载 mi unlock 工具 [unlock](http://www.miui.com/unlock/index.html)，登陆好 unlock 工具，关机手机，之后 音量“-”+ 开机键，进入 fastboot 模式。连接电脑。使用 unlock 工具进行解锁。



## 刷入 twrp rec

下载 mido 对应的 [twrp img](https://dl.twrp.me/mido/)，在获取 Android SDK 组件 [platform-tools](https://developer.android.com/studio/releases/platform-tools)。

手机连接电脑后，执行 ```  adb reboot bootloader ``` ，这是手机上会提示“是否允许电脑调试”，选择允许。手机会自动重启进入 fastboot 兔子。

执行如下命令刷入 img。
```  
fastboot flash recovery twrp.img 
```

执行如下命令重启手机。之后，快速的断开手机电脑连接线，快速用 音量“+”+ 开机键，使手机进入第三方 REC，如果这一步不够快，可能 REC 就刷失败了。
```
fastboot reboot
```

## 刷入卡刷包

三清/四清/五清
选择 ZIP 包，刷机。
