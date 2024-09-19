# Windows Driver开发环境配置

## 安装VisualStudio+SDK+WDK环境

全部流程：https://learn.microsoft.com/zh-cn/windows-hardware/drivers/download-the-wdk

注意：WDK安装前要求先安装适配版本的SDK：

https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/

推荐按以上链接在VS安装程序中安装Windows 11 SDK (10.0.26100.0)，注意默认选中的不是这个版本，需要手动选择这个版本：

![image-20240812164012107](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202408121640199.png)

正常安装完SDK和WDK后，创建一个KMDF项目是像这样：如果缺少SDK和Driver Setting这些，说明SDK版本不匹配

![image-20240812164731134](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202408121647174.png)

注意：如果在有问题的环境创建了项目编译会报错，在环境配好后该项目也不能用还是会编译报错，应该删除重新建。

## 环境配置

### 使用VSCode快捷键

工具–>选项->键盘->键盘映射方案选VSCode

### WDF项目找不到头文件问题

1. 找不到<ntddk.h>和<wdf.h>

在项目配置-> C/C++ -> General -> Additional Include Directories -> 加上WDK和WDF的include头文件路径

Ntddk.h contains core Windows kernel definitions for all drivers, while Wdf.h
contains definitions for drivers based on the Windows Driver Framework (WDF).  

![image-20240812104158672](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202408121042763.png)

注意WDF的版本，Win11选WDF 1.33，参考：https://learn.microsoft.com/zh-cn/windows-hardware/drivers/wdf/kmdf-version-history

2. 找不到device.tmh

项目设置 -> WPP Tracing -> 设置 "Run Wpp Tracing" 为 YES

### WDF项目找不到链接symbol

https://learn.microsoft.com/en-us/cpp/error-messages/tool-errors/linker-tools-error-lnk2019?view=msvc-170#third-party-library-issues-and-vcpkg

如果是调用第三方库API报此问题，基本上是项目配置没有链接这个库

The object file or library that contains the definition of the symbol isn't linked

以SDBUS驱动为例，ntddsd.h定义的SdBusSubmitRequest只有declaration，其函数体实现其实是在SDBUS.lib里，用everything搜索此lib（WDK路径），加到项目配置的Link dependence lib（Link -> Input -> Additional Dependence），即可编译通过。

The *ntddsd.h* header file, which is provided in the Windows Driver Kit (WDK), declares the prototypes for the routines exposed by the SD bus library.

https://learn.microsoft.com/en-us/windows-hardware/drivers/sd/sd-card-driver-stack

![image-20240822195016239](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202408221950345.png)

另外一个示例：

RtlStringCbVPrintfA打印函数属于ntstrsafe.h定义，其lib同名，位于WDK的km目录；一般WDF驱动把km和kmdf的.lib都加到项目的linker路径：（Link -> Input -> Additional Dependence）

注意需要指定到.lib文件名，一般是用到哪个lib才链接哪个lib；

```
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\km\x64\ntstrsafe.lib
```

也可以用*匹配所有lib：

```
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.26100.0\km\x64\*.lib

C:\Program Files (x86)\Windows Kits\10\Lib\wdf\kmdf\x64\1.33\*.lib
```

