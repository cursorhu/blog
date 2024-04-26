---
title: SD host controller的PCIe协议包分析
date: 2023-11-22 14:39:20
tags: PCIe
categories: debug
---

本文记录用LeCroy PCIe协议分析仪debug SD host controller issue的方法。具体背景是该issue使用windbg或者打开driver debug log后无法复现，只能用PCIe协议分析仪分析。

### PCIe包解析的配置

使用spread view模式：

![image-20231122144912104](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221449131.png)

数据解析方式使用小端（Little-endian）+ MSB（高bit解析）


### 寻找目标PCIe设备

本文的PCIe设备为SD host controller。

以下PCIe trace包含一次Memory read和Config read操作，以Memory read为例，一次通信流程是：

MRd 指定Address -> MRd ACK -> CpID包含设备返回的数据（98601217）-> CpID的ACK 

![image-20231122144334835](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221443875.png)

一般PCIe包开头会枚举设备，此CpID数据即目标设备的VendorID/DeviceID，反推出此设备在PCIe config space的mapping起始地址是0x51200000

那么SD host设备的register被mapping到哪里？这是由PCIe config space的BAR0/BAR1/...地址指定。

如下图，SD host controller register space起使于PCIe config space的0x51200000，其中的0x10~0x27 offset内的值即BAR0/BAR1/...空间的地址

![image-20231122143946372](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221439448.png)

看PCIe包即可得到SD host controller的Bar0位于0x51201000，即PCIe config space的4KB位置。该空间包含SD host register(由该chip的设计决定)

### SD command分析

既然已知SD host register，那么查找SD command，实际就是查找对SD command register (如下图0Eh)的Memory write操作，根据写入数据查看SD command index。

![image-20231122150434747](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221504817.png)

具体包分析如下：

下图0x5120100C中的0x0C即SD host register 0x0C，其写入数据Dword的高16bit即0x0E，所以最高byte即代表command index，即0x19是SD Command25: 多block写。

![image-20231122151211580](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221512611.png)

![image-20231122151622555](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221516595.png)

如何查看该SD command是否正常完成：

查看中断状态register 0x30h, 一般正常完成会返回command complete/ transfer complete等正常状态；未完成一般有Error interrupt。

![image-20231122151913284](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221519332.png)

对于error interrupt，在0x32（即0x30高16bit）查看具体类型：

![image-20231122152004564](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221520612.png)下面两个CpID分别是正常返回和错误返回：

![image-20231122152111371](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221521406.png)

![image-20231122152045475](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202311221520509.png)

正常返回的0x32(0x30高16bit)全为0，错误返回非0；错误类型为ADMA error + CRC error (data 0x32 = 0x0220)
