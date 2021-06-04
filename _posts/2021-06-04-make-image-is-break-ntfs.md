---
layout: post
title: "制作一个无法mount ntfs 镜像，但是能被ntfsfix修复"
subtitle: 'make image, it is break ntfs'
author: "cslqm"
header-style: text
tags:
  - OS
---

镜像大师又遇到高难度问题了，需要制作一个镜像：镜像中的 ntfs 不能 mount，但是呢，用 ntfsfix 能修复，修复后，又能 mount。

下边是这种尝试的记录，每一个大尝试是一个二级标题。

## 磁盘休眠的镜像无法mount

### 来源：https://askubuntu.com/questions/384429/cannot-remove-hiberfile-on-ntfs-partition

### 此方案遇到的问题：
windows 虚机内部找不到休眠的地方（按钮/配置项）。
powercfg /a 查看显示 S0~S3的各种固件不支持休眠操作。
尝试修改 libvirt xml，增加 pm suspend-to-disk，但是启动虚机后，powercfg 老样子。
发现libvirt文档中，有提到虚机 enable BIOS support S3 (suspend-to-mem) and S4 (suspend-to-disk) ACPI sleep states。

所以libvirt xml 
``` xml
  <pm>
    <suspend-to-mem enabled='yes'/>    <suspend-to-disk enabled='yes'/>   </pm>
```
启动虚机后，进入电源管理。修改电源其他配置，打开“快速关机”功能，打开“显示睡眠”菜单功能。

关机。成功实现镜像无法mount。
``` sh
mount /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/
mount: you must specify the filesystem type
```

### 结果：
做的镜像确实不能 mount了，但是ntfsfix 修复不了了，也不能mount。。
``` sh
$ ntfsfix  /dev/nbd15p2 
Mounting volume... Windows is hibernated, refused to mount.
FAILED
Attempting to correct errors... 
Processing $MFT and $MFTMirr...
Reading $MFT... OK
Reading $MFTMirr... OK
Comparing $MFTMirr to $MFT... OK
Processing of $MFT and $MFTMirr completed successfully.
Setting required flags on partition... OK
Going to empty the journal ($LogFile)... OK
Windows is hibernated, refused to mount.
Remount failed: Operation not permitted

$ mount /dev/nbd15p2  /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/
Windows is hibernated, refused to mount.
Failed to mount '/dev/nbd15p2': Operation not permitted
The NTFS partition is in an unsafe state. Please resume and shutdown
Windows fully (no hibernation or fast restarting), or mount the volume
read-only with the 'ro' mount option.
```


## 通过工具修改磁盘MFT数据

DiskGenius V5.4.2

运行，选择进入 WINpe模式
选择 C 盘
16进制编辑模式
随便找个地方改改 将 NTFS head $MFT  cluster number 加个2 0x30 长度 8bytes，最后的字节 00->02 

https://en.wikipedia.org/wiki/NTFS

### 结果
系统都起不来了，文件系统依然能挂载。。。



## 破坏分区数据

ntfs分区在磁盘分布

``` txt
---------------------------------------------------------------
|MBR和空白-1MB| DBR  分区数据   DBR备份｜ DBR  数据  DBR 备份｜ ...
---------------------------------------------------------------
```

但是 qcow2 镜像本身存在l1/l2两级cluster管理，需要计算偏移量。不如直接convert raw镜像。

``` sh
$ qemu-img  convert -f qcow2 -O raw /opt/data/limg/03dc671d-7faf-487c-9810-d03558b4bf9c.img.out  /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img
```

使用virt-filesystem 查看镜像内部分区情况
``` sh
$ virt-filesystems --long -a /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img --all
Name      Type       VFS  Label        MBR Size        Parent
/dev/sda1 filesystem ntfs 系统保留 -   524288000   -
/dev/sda2 filesystem ntfs -            -   42423287808 -
/dev/sda1 partition  -    -            07  524288000   /dev/sda
/dev/sda2 partition  -    -            07  42423287808 /dev/sda
/dev/sda  device     -    -            -   42949672960 -
```

所以，可以将 sda1（保留分区） 和 sda2（c盘）中间部分数据进行修改，直接修改 sda2 分区的 DBR。使其文件系统结构损坏。

``` sh
$ dd if=/dev/zero  of=/opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img seek=524088000  obs=1 count=5 ibs=100000000
5+0 records in
500000000+0 records out
500000000 bytes (500 MB) copied, 657.247 s, 761 kB/s
```

顺便把 MBR 分区搞了
``` sh
$ dd if=/dev/zero  of=/opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img seek=350  obs=1 count=1 ibs=100
```

做完后，镜像已经失去了原有的基本数据信息了，虚拟磁盘大小都没有了
``` sh
$ qemu-img  info /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img 
image: /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img
file format: raw
virtual size: 977M (1024088064 bytes)
disk size: 908M
```

搞成这个样子了，居然还能挂 sda2。。
``` sh
$ qemu-nbd -c /dev/nbd15  /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img
WARNING: Image format was not specified for '/opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operations on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
Failed to set NBD socket
Disconnect client, due to: End of file

$ ps -ef | grep /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img
root     11053     1  0 Jun03 ?        00:00:00 qemu-nbd -c /dev/nbd15 /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img
root     40386  2508  0 09:54 pts/5    00:00:00 grep /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img

$ mount /dev/nbd15p2  /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/

$ ls /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/
bootmgr  Documents and Settings  PerfLogs     Program Files        rearm.flag  $Recycle.Bin               Users
BOOTNXT  pagefile.sys            ProgramData  Program Files (x86)  Recovery    System Volume Information  Windows
```

### 结果：

当然是失败了

## 尝试删除NTFS文件系统分区相关数据

ntfsfix 修复mount的代码
``` C
/**
 * fix_mount
 */
static int fix_mount(void)
{
	int ret = 0; /* default success */
	ntfs_volume *vol;
	struct ntfs_device *dev;
	unsigned long flags;

	ntfs_log_info("Attempting to correct errors... ");

	dev = ntfs_device_alloc(opt.volume, 0, &ntfs_device_default_io_ops,
			NULL);
	if (!dev) {
		ntfs_log_info(FAILED);
		ntfs_log_perror("Failed to allocate device");
		return -1;
	}
	flags = (opt.no_action ? NTFS_MNT_RDONLY : 0);
	vol = ntfs_volume_startup(dev, flags);
	if (!vol) {
		ntfs_log_info(FAILED);
		ntfs_log_perror("Failed to startup volume");

		/* Try fixing the bootsector and MFT, then redo the startup */
		if (!fix_startup(dev, flags)) {
			if (opt.no_action)
				ntfs_log_info("The startup data can be fixed, "
						"but no change was requested\n");
			else
				vol = ntfs_volume_startup(dev, flags);
		}
		if (!vol) {
			ntfs_log_error("Volume is corrupt. You should run chkdsk.\n");
			ntfs_device_free(dev);
			return -1;
		}
		if (opt.no_action)
			ret = -1; /* error present and not fixed */
	}
		/* if option -n proceed despite errors, to display them all */
	if ((!ret || opt.no_action) && (fix_mftmirr(vol) < 0))
		ret = -1;
	if ((!ret || opt.no_action) && (fix_upcase(vol) < 0))
		ret = -1;
	if ((!ret || opt.no_action) && (set_dirty_flag(vol) < 0))
		ret = -1;
	if ((!ret || opt.no_action) && (empty_journal(vol) < 0))
		ret = -1;
	/*
	 * ntfs_umount() will invoke ntfs_device_free() for us.
	 * Ignore the returned error resulting from partial mounting.
	 */
	ntfs_umount(vol, 1);
	return ret;
}
```

从代码中呢，可以发现，主要修复的就是 $MFT $MFTMirr $UpCase 这几个NTFS关键字段，这些东西在ntfs中时文件形式存在的，都在各个分区的根目录，sda2（nbdXp2）的也就是 C:\。


一 DiskGenius 修改不了。 winPE模式进不去，DOS模式支持的功能太少，无法编辑。

二 对 MFT 创建硬链接，删除链接文件。 失败，MFT文件mklink命令不可读。

三 破坏分区 head 部分4k数据使其找不到 MFT 

``` sh
$ qemu-nbd -c /dev/nbd15  /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img

$ dd if=/dev/zero  of=/dev/nbd15p2   count=1 bs=4096
1+0 records in
1+0 records out
4096 bytes (4.1 kB) copied, 0.225662 s, 18.2 kB/s

$ mount /dev/nbd15p2  /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/  
mount: you must specify the filesystem type


$ qemu-nbd -d /dev/nbd15
```

基于上边做出的问题镜像，create新镜像测试。
``` sh
$ qemu-img  create -f qcow2 -b /opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img test.img
Formatting 'test.img', fmt=qcow2 size=42949672960 backing_file=/opt/data/limg/464c76f0-1d0c-44f3-979f-279eb7a3bd58.img cluster_size=65536 lazy_refcounts=of
f refcount_bits=16

$ qemu-nbd -c /dev/nbd15  test.img  

$ mount /dev/nbd15p2  /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/
mount: you must specify the filesystem type

$ ntfsfix  -d /dev/nbd15p2 
Mounting volume... NTFS signature is missing.
FAILED
Attempting to correct errors... NTFS signature is missing.
FAILED
Failed to startup volume: Invalid argument
NTFS signature is missing.
Trying the alternate boot sector
The alternate bootsector is usable
Rewriting the bootsector
The boot sector has been rewritten

Processing $MFT and $MFTMirr...
Reading $MFT... OK
Reading $MFTMirr... OK
Comparing $MFTMirr to $MFT... OK
Processing of $MFT and $MFTMirr completed successfully.
Setting required flags on partition... OK
Going to empty the journal ($LogFile)... OK
Checking the alternate boot sector... OK
NTFS volume version is 3.1.
NTFS partition /dev/nbd15p2 was processed successfully.


$ mount /dev/nbd15p2  /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/  

$ ls /mnt/464c76f0-1d0c-44f3-979f-279eb7a3bd58/
bootmgr  Documents and Settings  ProgramData    Program Files (x86)  $Recycle.Bin               Users
BOOTNXT  pagefile.sys            Program Files  Recovery             System Volume Information  Windows
```

### 结果

果然还是 qemu-nbd 好用。任务完成。
