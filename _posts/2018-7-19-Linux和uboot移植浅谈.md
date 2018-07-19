---
layout:     post
title:      Linux系统完整移植浅谈
subtitle:   Linux移植
date:       2018-7-19
author:     Muggle
header-img:
catalog: 	 true
tags:
    - LINUX之Linux移植
    - LINUX之uboot移植
    - LINUX之构建根文件系统
---

今天一天移植了LINUX3.4.2和uboot2012.4对LINUX系统移植这方面进行了简单的体验，使用的根文件系统busybox1.7.0也比较老，所以后面将在JZ2440上面移植最新的LINUX系统和UBOOT。下面简单说说今天一天的收获。<br>
首先是解压`linux-3.4.2.tar.bz2`和`u-boot-2012.04.01.tar.bz2`分别进入解压后的文件夹，使用`patch -p1 < ../xxx.patch`命令打上补丁文件，UBOOT和LINUX制作补丁的方法参考博客[https://www.cnblogs.com/kele-dad/category/1009200.html](https://www.cnblogs.com/kele-dad/category/1009200.html)裁剪内核也参考这个博客<br>
1.制作`uboot.bin`文件，输入`make smdk2440_config`生成`.config`文件，再执行`make`然后等编译完目录下就有了`uboot.bin`。使用openjtag将其下载到板上
2.制作uImage文件，`make mini2440_defconfig`,`make menuconfig`裁剪掉冗余的功能，`make uImage`生成内核映像。然后使用TFTP方式下载，构建TFTP不再说<br>
TFTP下载内核：
	tftp 0x30007FC0 uImage
	nand erase.part kernel
	nand write 0x30007FC0 kernel
TFTP下载文件系统jffs2/yaffs2:
	tftp 32000000 fs_mini_mdev.yaffs2/jffs2
	nand erase.part rootfs
	nand write.yaffs 32000000 0x00460000 889bc0
	nand write.jffs2 30000000(base) 0x00460000(offet) 5b89a8(size)

问题：在下载完成后，启动单板，会出现各种错误，大部分都和裁剪内核的时候有些选项没有选上或者多选了有关系。这里不再一一描述，根据不同的情况进行百度解决。

