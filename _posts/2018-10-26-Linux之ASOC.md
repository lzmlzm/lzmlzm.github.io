---
layout:     post
title:      ASOC分析
subtitle:   Linux驱动
date:       2018-10-26
author:     Muggle
header-img:
catalog: 	 true
tags:
    - Linux驱动
---

## 概述 ##
ASLA存在的问题： <br>
1.  Codec驱动与SOC中断CPU耦合严重，这将导致代码重复，一个Codec驱动每个cpu上会出现不同的版本。<br>
2.  当音频事件发生时(插拔耳机，音箱)没有标准的方法通知用户，尤其在移动端此事件非常常见。<br>
3.  当播放/录制音频时，驱动会让整个codec处于上电状态，这样会在移动端非常浪费电量。同时也不支持改变采样频率/骗置电流来节约功耗。<br>

ASOC的解决办法：<br>

1.  Codec代码独立，不再耦合与CPU，这样可以增加Codec代码重复利用。<br>
2.  在Codec和Soc之间通过简单的I2S/PCM音频接口通信，这样SOC和Codec只需要注册自己相关的接口到ASOC code即可。<br>
3.  动态的电源管理(Dynamic Audio Power Management)DAPM。DAPM始终将Codec自动设置在最低功耗状态运行。<br>
4.  消除pop音。控制各个widget上下电的顺序消除pop音。<br>
5.  添加平台相关的控制，运行平台添加控制设备到声卡。<br>

## ASOC架构 ##
ASOC将嵌入式音频系统分为三大类可重复使用的驱动程序:  Platform,  Machine,  Codec。<br>

Codec类:     Codec即编解码芯片的驱动，此Codec驱动是和平台无关，包含的功能有:  音频的控制接口，音频读写IO接口，以及DAPM的定义等。如果需要的话，此Codec类可以在BT，FM，MODEM模块中不做修改的使用。因此Codec就是一个可重复使用的模块，同一个Codec在不同的SOC中可以使用。<br>

Platform类:  可以理解为某款SOC平台，平台驱动中包括音频DMA引擎驱动，数字接口驱动(I2S, AC97, PCM)以及该平台相关的任何音频DSP驱动。同样此Platform也可以重用，在不同的Machine中可以直接重复使用。<br>

Machine类:  Machine可以理解为是一个桥梁，用于在Codec和Platform之间建立联系。此Machine指定了该机器使用那个Platform，那个Codec，最终会通过Machine建立两者之间的联系。<br>
![](https://i.imgur.com/aCTCewV.jpg)
