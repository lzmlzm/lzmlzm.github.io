---
layout:     post
title:      buildroot构建s3c2440步骤
subtitle:   buildroot
date:       2018-7-12
author:     Muggle
header-img:
catalog: 	 true
tags:
    - buildroot
---


**要构建自己的开发板，首先要创建一个基本的 buildroot配置作为开发板的基本编译系统。这里包括toolchain，kernel，bootloader，filesystem 和一个简单的 busy-box 用户空间。不要选择特别的配置，这个配置必须要足够小，并仅仅作为目标平台一个基本的 BusyBox 系统。**

　　执行 make menuconfig：

　　

    Target options：目标板的配置
        Target Architecture：目标架构，这里选择 ARM(little endian)，ARM小端模式
        Target Binary Format：二进制格式，为 ELF
        Target Architecture Variant：架构变体为 arm920t，内核类型
        Target ABI：应用程序二进制接口，为EABI
        Floating point strategy：浮点数的策略，选择为 Soft float
        ARM instruction set：arm 汇编指令集，选择  ARM
    Build options：主要是一些编译时用到的选项,比如dl的路径,下载代码包使用的路径,同时运行多个编译的上限,是否使能编译器缓冲区等等,这里按照默认就行了.
    Toolchain：工具链选项
        Toolchain type：Buildroot提供两种方式使用toolchain
            external toolthain：非Buildroot提供的交叉编译器
            Buildroot toolchain：Buildroot本身编译生成的Buildroot toolchain，直接选择此项
        custom toolchain vendor name：填上S3C2440
        C library：C库选择，选择 glibc
        Kernel Headers：内核头文件，Linux 4.9.x kernel headers
        glibc version：glibc版本选择，2.24
        Binutils Version：binutils版本：2.27
        Additional binutils options：附加的 binutils 选择，不填即可
        GCC compiler Version：GCC版本选择，gcc 6.x
        Additional gcc options：附件的GCC选项，不填写即可
        Enable C++ support：使能C++支持，选上
        Enable Fortran support：使能Fortran语言支持，不选
        Enable compiler link-time-optimization support：是否支持LTO，不选，LTO是什么：http://blog.csdn.net/fickyou/article/details/52381776
        Enable compiler OpenMP support：支持OpenMP？OpenMP用于共享内存并行系统的多处理器程序设计，OpenMP并不适合需要复杂的线程间同步和互斥的场合，OpenMp的另一个缺点是不能在非共享内存系统(如计算机集群)上使用。不选择
        Enable graphite support ：是否支持graphite。Graphite是应用WEB应用的一套开源的编程接口。不选择。具体看百度百科：https://baike.baidu.com/item/Graphite/9810474?fr=aladdin
        Build cross gdb for the host：主机上运行gdb进行调试，不选
        Copy gconv libraries：拷贝 gconv库，gconv库用于在不同字符集之间进行转换。默认不选即可
        Enable MMU support：使能 MMU，S3C2440支持MMU，选上
        Target Optimizations：不选
        Target linker options：不选
        Register toolchain within Eclipse Buildroot plug-in：eclipse插件支持，不选
    System configuration：系统配置
        Root FS skeleton：
        System hostname：填写JZ2440　
        System banner
        Passwords encoding
        Init system：系统初始化，选择 BusyBox
        /dev management：设备文件管理，选择Dynamic using devtmpfs + mdev，即使用mdev动态加载设备节点的方式
        Path to the permission tables：设备节点的配置表设置，一定要选择system/device_table_dev.txt，否则后面在dev目录下将不会生成各种设备节点。当然我们也可以手动的配置该文件，添加必要的节点或删除不需要的节点。
        support extended attributes in device tables
        Use symlinks to /usr for /bin, /sbin and /lib
        Enable root login with password
        Root password：进入linux控制台终端后的密码，为空则登录时不需要密码，默认登录用户名为root。为空。
        /bin/sh (busybox' default shell)
        Run a getty (login prompt) after boot：保持默认，默认为选中。
            TTY port：配置为 ttySAC3
            Baudrate ：波特率，配置为 115200
            TERM environment variable：默认即可
            other options to pass to getty：默认即可
        remount root filesystem read-write during boot
        Network interface to configure through DHCP
        Purge unwanted locales
        Locales to keep
        Generate locale data
        Install timezone info
        Path to the users tables
        Root filesystem overlay directories：
        Custom scripts to run before creating filesystem images
        Custom scripts to run inside the fakeroot environment
        Custom scripts to run after creating filesystem images
    Kernel：内核配置
        Kernel version：内核版本，选择用户自定义，Custom version
        Kernel version：填上自己所需要的版本，4.14.12
        Custom kernel patches：自定义的内核补丁，无
        Kernel configuration：内核配置，选择 Using an in-tree defconfig file
        Defconfig name：填写为 mini2440
        Additional configuration fragment files：暂且不填写
        Kernel binary format：内核二进制文件格式，zImage
        Kernel compression format：内核压缩格式，选择gzip即可
        Build a Device Tree Blob：设备树？暂且不填写
        Install kernel image to /boot in target：暂且不填
        Linux Kernel Extensions：内核扩展，默认不选择
        Linux Kernel Tools：内核工具，默认不选择
    Target packages
    Filesystem images：文件系统选择，选择 yaffs2 root filesystem
    Bootloaders：硬件启动程序，选择为 U-boot
        Build system：u-boot系统选择为Kconfig
            legacy：若是选择2015.04之前的u-boot 选择此项
            Kconfig：2015.04之后的 u-boot 选择此项，勾选此项　　
        U-boot Version：U-boot版本，默认为 2017.01，选择为Custom version
        U-Boot version：填写为2017.11
        Custom U-boot patches：U-boot补丁，不添加
        U-Boot configuration：U-boot配置，暂时还没有U-BOOT，所以选择为：Using an in-tree board defconfig file
        Board defconfig：板子的配置，选择与架构一样的板子的默认文件，mini2440。后期再修改
        U-boot needs dtc：是否需要设备树，默认，后期调试
        U-boot needs OpenSSL：是否需要 OpenSSL，默认，后期调试修改
        U-boot binary format：二进制文件，选择 .bin文件
        produce a .ift signed image：默认
        Install U-boot SPL binary image：默认
        Environment image：默认
    Host utilities
    Legacy config options

　　**配置完成后，执行make命令。**
更多详细请参考[http://www.cnblogs.com/kele-dad/p/8231434.html](http://www.cnblogs.com/kele-dad/p/8231434.html)