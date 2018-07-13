---
layout:     post
title:      Linux驱动-USB驱动(一）概念模型篇
subtitle:   Linux驱动
date:       2018-7-15
author:     Muggle
header-img:
catalog: 	 true
tags:
    - Linux驱动
---

编写与一个USB设备驱动程序的方法和其他总线驱动方式类似，驱动程序把驱动程序对象注册到USB子系统中，稍后再使用制造商和设备标识来判断是否安装了硬件。当然，这些制造商和设备标识需要我们编写进USB 驱动程序中。USB 驱动程序依然遵循设备模型 —— 总线、设备、驱动。
# 一、注册USB驱动程序
和其他的驱动程序一样，USB驱动程序也要编写入口函数和出口函数，并且加以修饰，加上GPL。
### 首先定义一个`usb_driver`的结构体
![](https://i.imgur.com/cUC0LgC.jpg)
### 然后进行入口函数的注册和出口函数里的销毁
![](https://i.imgur.com/aUFbarS.jpg)

由图一可以看出还需要完成`.probe`和`.disconnect`函数的编写。
# 二、.probe函数的编写
和输入子系统类似，先列下注释：
	 1./* 分配一个input_dev结构体 */
	 2./* 设置 */
	 3./* 能产生哪类事件 */
	 4./* 能产生哪类中的哪些事件 */
	 5./* 注册 */
	 6./* 硬件相关 */
1.首先实例化一个`input_dev`结构体`uk_dev`,并且使用输入子系统里面的`input_allocate_device()`进行分配。
3.设置能产生哪类事件`set_bit()`，`EV_KEY`按键类事件，`EV_REP`重复性事件，第二个参数指向实例化结构体的事件类
4.设置能产生哪类事件中的哪些事件
5.硬件相关，数据传输的三要素①源：USB的某个端点②传输长度③传输的目的地④申请USB请求块⑤使用三要素设置uk_urb⑥提交uk_urb
# 三、.disconnect函数的编写
1.杀掉`uk_urb`进程<br>
2.释放`uk_urb`
3.释放缓冲区
4.销毁注册的输入设备结构体
5.释放设备

![](https://i.imgur.com/tcw9bcv.jpg)
# 四、`usbmouse_as_key_id_table`的编写
直接复制参考驱动
![](https://i.imgur.com/YTekgvD.jpg)
# 五、鼠标中断程序
判断数据的每一位代表的含义，并且进行事件上报和同步，实现鼠标键盘的功能。
![](https://i.imgur.com/UelSfVd.jpg)