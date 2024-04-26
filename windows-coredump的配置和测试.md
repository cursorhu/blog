---
title: windows coredump的配置和测试
date: 2023-07-24 15:42:43
tags: windows
categories: windows
---

在存储设备的windows系统环境下调试时，因为存储设备本身的问题，有时候coredump不能成功生成到系统目录，本文记录如何修改coredump路径，以及用键盘测试coredump生成符合预期。

## 使能windows的coredump
[Enabling a Kernel-Mode Dump File](https://learn.microsoft.com/zh-CN/windows-hardware/drivers/debugger/enabling-a-kernel-mode-dump-file)

## 修改coredump路径
[Windows 的内存转储文件选项概述](https://learn.microsoft.com/zh-cn/troubleshoot/windows-server/performance/memory-dump-file-options)

可以修改注册表`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\CrashControl`的DumpFile键值对，默认路径%SystemRoot%在cmd echo出来是"C:\", 修改为指定路径例如"E:\Memory.dmp"。

![image-20230724155525547](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202307241555783.png)

此操作也可以在控制面板完成，两者等效。

![crashcontrol2](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202307241555568.PNG)

## 使用键盘手动生成coredump

[Forcing a system crash from the keyboard](https://learn.microsoft.com/zh-CN/windows-hardware/drivers/debugger/forcing-a-system-crash-from-the-keyboard)

以USB keyboards为例：

修改注册表`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\kbdhid\Parameters`，创建CrashOnCtrlScroll = 0x01，重启后“Hold down the rightmost CTRL key, and press the SCROLL LOCK key twice.”系统会直接蓝屏，重启即可查看coredump文件。如果用windbg查看KeBugCheck查看错误码是0xE2: MANUALLY_INITIATED_CRASH。

![image-20230724155442525](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202307241554830.png)

