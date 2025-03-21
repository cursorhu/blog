Cursor composer记录 -- Windows driver development for SD Host.

# SDHC WDF driver开发

## 请分析 @sdstor 目录的storport miniport SD host driver代码of O2micro, 并分析用WDM还是WDF重写更合适

```
让我分析一下这个O2micro的SD Host Controller驱动:

代码架构分析:

采用分层架构:

card层: 处理SD/MMC/eMMC/UHS2卡的初始化和命令处理

host层: 处理主机控制器寄存器访问和中断处理

tagqueue层: 处理命令队列和DMA传输

util层: 提供调试和工具函数

main层: 处理电源管理、线程和请求管理等

使用Storport miniport模型,实现了SCSI miniport接口

主要特点:

支持多种卡类型(SD/MMC/eMMC/UHS2)

支持多种DMA模式(SDMA/ADMA2/ADMA3)

支持标签队列和命令合并

支持电源管理和热插拔

有完善的调试和错误恢复机制

重写建议:

我建议使用WDF重写,原因如下:

优点:

WDF提供更高级的抽象,可以简化驱动开发

WDF自动处理PnP和电源管理,减少代码量

WDF提供内置的同步机制,更容易处理并发

WDF有更好的调试支持

WDF框架更现代,维护性更好

需要注意的地方:

需要重新设计请求队列,使用WDF的队列对象

DMA传输需要使用WDF的DMA框架

中断处理需要适配WDF的中断对象

可能需要重构部分硬件访问代码

建议的重构步骤:

创建WDF驱动框架

将硬件访问代码迁移到WDF回调函数

使用WDF队列替换tagqueue层

使用WDF DMA框架重写DMA传输

适配中断处理和电源管理

保留card层的卡管理逻辑

重用util层的调试功能

总的来说,使用WDF重写可以简化代码,提高可维护性,同时保留现有的卡管理和调试功能。虽然需要投入一定的重构工作,但长期来看是值得的。

让我知道如果你需要更详细的分析或具体的重构建议
```

## SD卡的盘符如何才能显示在文件管理器上

```
要让SD卡在Windows文件管理器中显示为可访问的盘符，需要实现PnP设备接口和存储类接口。我们需要添加以下功能：

首先在driver.h中添加存储相关定义：

driver.h15/15

在driver.c中添加存储接口支持：

driver.c13/13

在INF文件中添加磁盘类支持：

o2micro_sdhost.inf4/5

这些修改将：

创建磁盘设备接口

处理必要的磁盘IOCTL请求

提供磁盘几何信息和容量信息

在卡插入时启用磁盘接口

在卡移除时禁用磁盘接口

Windows将通过这些接口识别SD卡为可移动磁盘，并在文件管理器中显示对应的盘符。用户可以像使用普通U盘一样访问SD卡。

需要注意的是，这里只实现了基本的磁盘接口。完整的实现还需要：

分区表支持

文件系统支持

读写缓存支持

安全弹出支持

电源管理集成
```

## 添加VisualStudio编译中对中文字符的支持

```
我来帮您添加对中文字符的支持。需要修改项目文件和源代码文件的编码设置。
修改项目文件：
在driver.h中添加编码声明：
在driver.c中添加编码声明：
在trace.h中添加编码声明：
这些修改：
在项目文件中：
添加了Unicode字符集支持
设置了UTF-8编码选项
添加了中文资源支持
在源代码文件中：
添加了UTF-8编码声明
修改了字符串为Unicode格式
更新了调试输出宏以支持Unicode
确保了所有字符串常量使用Unicode格式（L前缀）
这样修改后，项目应该能够正确处理中文字符，并且在Visual Studio中编译时不会出现中文乱码问题。记得在保存所有源代码文件时使用UTF-8编码（带BOM）。
```

## %!STATUS!能正确打印Status的值吗，为什么不用%x

```
%!STATUS!是Windows驱动开发中的一个特殊格式说明符，它是DbgPrint/DbgPrintEx的扩展功能，专门用于打印NTSTATUS值。它比%x更好，因为：

%!STATUS!会将NTSTATUS值转换为可读的字符串，例如：

STATUS_SUCCESS 会显示为 "STATUS_SUCCESS"

STATUS_INSUFFICIENT_RESOURCES 会显示为 "STATUS_INSUFFICIENT_RESOURCES"

而不是仅显示十六进制值

使用示例：

NTSTATUS status = STATUS_INSUFFICIENT_RESOURCES;

// 使用%!STATUS!

O2MICRO_LOG_ERROR("Operation failed: %!STATUS!", status);

// 输出: Operation failed: STATUS_INSUFFICIENT_RESOURCES

// 使用%x

O2MICRO_LOG_ERROR("Operation failed: 0x%x", status);

// 输出: Operation failed: 0xC000009A

其他类似的格式说明符：

%!HRESULT! // 用于HRESULT值

%!WINERROR! // 用于Windows错误码

%!GUID!   // 用于GUID

%!TIME!   // 用于时间戳

%!HEXDUMP! // 用于内存dump

所以我们应该继续使用%!STATUS!，因为它提供了更好的可读性和调试体验。
```

DbgPrint能支持%!STATUS!吗

```
不能，DbgPrint不支持%!STATUS!格式说明符，这是DbgPrintEx的特殊功能。让我修改调试宏：
主要修改：
将%!STATUS!改为0x%08X
使用标准的C格式说明符
保持其他功能不变
```

## DBG在哪里被定义

在Windows驱动开发中，`DBG`通常是在项目配置或编译选项中定义的。有以下几种常见的定义方式：

1. 在项目配置中：
- Debug配置会自动定义`DBG=1`
- Release配置则不会定义`DBG`

2. 在源代码中，你可以在driver.h文件的最开头添加如下定义：

```c:driver.h
#ifndef DBG
#if defined(DEBUG) || defined(_DEBUG)
#define DBG 1
#else
#define DBG 0
#endif
#endif
```

建议在driver.h文件的最开头添加这段代码，在`#pragma once`之后即可：

```c:driver.h
#pragma once
#pragma code_page(65001)  // UTF-8编码

// 添加DBG定义
#ifndef DBG
#if defined(DEBUG) || defined(_DEBUG)
#define DBG 1
#else
#define DBG 0
#endif
#endif

// 其他代码保持不变...
```

这样可以确保：
1. 在Debug配置下启用调试输出
2. 在Release配置下禁用调试输出
3. 与Visual Studio的默认行为保持一致

## 修改LINK的inherited value有错误的方法

https://learn.microsoft.com/zh-cn/windows-hardware/drivers/develop/building-drivers-for-different-versions-of-windows

项目文件 (*.vcxproj)。 在项目文件中，找到驱动程序支持的配置的 **<PropertyGroup>** ，并添加要重写的宏来重写默认的链接器选项，如下重写了$(KernelBufferOverflowLib)

```
<KernelBufferOverflowLib>$(DDK_LIB_PATH)\BufferOverflowK.lib</KernelBufferOverflowLib>
```

## O2Micro的PCIe SD Host controller驱动，属于什么设备class，应该使用storport，WDF，还是WDM

对于O2Micro的PCIe SD Host controller驱动，我们需要分析：

设备特性：

PCIe接口的SD主机控制器

处理SD卡的存储功能

需要DMA传输支持

需要中断处理

2. 可选的驱动模型：

Storport：主要用于SCSI/SAS/RAID控制器

WDF：更现代的驱动框架，简化驱动开发

WDM：较底层的驱动模型

3.推荐选择：WDF (KMDF)，原因：

SD主机控制器是标准设备类(SDHost)

Windows有标准的SD总线支持

WDF提供了良好的PnP和电源管理支持

WDF简化了DMA和中断处理

不需要Storport的复杂SCSI功能

4.设备类设置：

[Version]
Class=SDHost
ClassGuid={a0a588a4-c46f-4b37-b7ea-c82fe89870c6}
