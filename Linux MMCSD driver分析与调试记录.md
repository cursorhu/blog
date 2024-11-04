---
title: Linux MMC/SD driver分析与调试记录
date: 2022-03-25 11:21:00
tags: linux
categories: linux
---

## 1. Linux MMC 框架现状
Linux MMC driver是支持包括SD卡，eMMC卡等等，属于MultiMediaCard设备和接口的驱动
其源码路径位于Kernel source code的drivers/mmc路径, 头文件位于include/linux/mmc
mmc源码分为core/host两层，是为了解耦：
- 通用的SD/eMMC流程(core)
- 具体的硬件操作流程(host)，在此层又可分为通用的SDHCI框架和非SDHCI框架，各eMMC/SD host厂商实现最底层driver时，可以遵循SDHCI框架下的API, 间接实现core层定义的方法(driver称为operations), 也可以不遵循SDHCI框架，直接实现core层定义的方法。
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301058425.png)

本文重点关注mmc框架对SD卡驱动的支持

### 1.1 SD卡的类型概述

SD卡可以分为三种类型：
UHS-I, UHS-II, SD express

详细信息参考https://www.sdcard.org
- Physical Layer Specification Ver.7.10 (从各层描述SD 7.0, SD 4.0, 以及更早版本SD的规范)
- SD Host Controller Specification Ver7.0 (从host控制器角度，描述SD 7.0, SD 4.0, 以及更早版本SD的规范)
- SD_Specifications_Part_1_UHS_II_Addendum(描述SD UHSII的附录规范)

UHS即Ultra High Speed, express也表示高速，这三代SD卡的读写速度是依次增加，参考下图：
- UHSI：50~104MB/s
- UHSII: 156~624MB/s
- SD express: 985MB/s

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301119947.png)
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301120184.png)

### 1.2 Linux MMC框架对SD卡的支持
基本概念：只有mmc框架的core层支持某种SD模式，host层才能实现这种模式；如果core层都不支持，只能厂商自己开发core层，以patch补丁的方式发布。

core层对于上述三种SD模式的支持：
- Linux kernel 5.11 以前，只支持UHS-I及其更低速度的legacy-SD模式
- Linux kernel 5.11 开始，在core层添加了SD express的支持
- 目前没有UHS-II的支持，只有提交待审核的，参考：[lore.kernel.org/Jason Lai/patch](https://lore.kernel.org/all/?q=Jason%20Lai)

host层对于上述三种SD模式的支持：
- UHS-I: 基本host目录的大多数SD厂商驱动都支持，很多符合sdhci框架
- SD express: Realtek基于Linux kernel 5.11的core层API, 实现了 驱动的host底层部分，参考kernel的host/rtsx_pci_sdmmc.c, 其没有使用SDHCI框架。
- UHS-II: 只有以patch方式实现的，参考[# linux-uhs2-gl9755](https://gitlab.com/ben.chuang/linux-uhs2-gl9755/)，其实现了core/host-sdhci/host vendor多个层次的UHS-II支持。

综上所述，本文参考uhs2-gl8755 patch，实现自己的SD UHSII driver。

## 2. 编译过程
本节描述编译mmc driver module和整个kernel的过程，同时描述中间踩的坑。
### 2.1 直接编译整个Kernel(带UHS-II Patch)
安装Linux Ubuntu 20版本，Ubuntu环境下载和解压待编译的整个Linux kernel 源码：[# linux-uhs2-gl9755](https://gitlab.com/ben.chuang/linux-uhs2-gl9755/)

注意：一定要在Linux环境下解压待编译源码，不能在windows下解压再拷到Linux编译，因为源码中有些大小写不同的同名文件，例如net/netfilter的很多头文件。windows不区分大小，解压时写会让你替换或重命名，这些同名文件的内容不一样，所以不能替换或重命名，强行替换会导致编译Linux报错找不到相关文件。

1. 编译环境准备
gcc/make等工具，都需要先安装build-essential等工具才能使用
```
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev
```

- 遇到的问题:

apt如果有依赖问题，建议apt手动安装，如果要特定版本，例如指定依赖libc6库版本为2.35-0ubuntu3，使用`apt install libc6=2.35-0ubuntu3`, 可以用apt policy libc6查看。

这里不建议sudo apt install aptitude（使用aptitude自动安装需要的依赖库版本），因为会导致make menuconfig出现<sys/types.h>找不到的问题，这个问题的原因是libc6-dev未安装，必须用apt安装libc6-dev解决此问题。

2. 配置，编译和安装 

```
cd linux-uhs2-gl9755-v3-patch #进入待编译Kernel源码
make menuconfig #配置内核，生成.config文件
make -j4 #以4线程编译内核，等同于make bzImage，make modules
make modules_install #安装各Driver模块
make install #安装内核(包括更新模块信息)
```
编译完成后会自动update-grub, 重启后选择编译好的kernel版本启动。

也可以设置默认启动的kernel，编辑/etc/default/grub的`GRUB_DEFAULT="1>X"`, 其中1表示从advanced选项启动，X表示从哪个kernel启动(0 based)，例如下图如果默认要从5.19启动，X设置为0，默认从5.8.0-rc4启动，X设置为6.
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202208171414943.png)
配置完毕必须要update-grub重启生效

- 遇到的问题

  make有canonical-certs.pem证书问题：修改.config，取消证书要求：CONFIG_SYSTEM_TRUSTED_KEYS=""，CONFIG_SYSTEM_REVOCATION_KEYS=""

3.  查看内核版本
```
uname -r #查看当前运行的kernel版本
cat Makefile #查看待编译kernel源码的内核版本
```
以linux-uhs2-gl9755-v3-patch为例，其根目录Makefile如下，表示kernel源码版本为 5.8.0-rc4
编译完成重启后应该选择5.8.0-rc4启动，进入桌面后用`uname -r`查看
```
VERSION = 5
PATCHLEVEL = 8
SUBLEVEL = 0
EXTRAVERSION = -rc4
NAME = Kleptomaniac Octopus
```

4. 编译报错记录
(1) 生成vmlinux Image时报错：
```
Failed to generate BTF for vmlinux  
Try to disable CONFIG_DEBUG_INFO_BTF
```
修改Kernel源码根目录的.config文件，CONFIG_DEBUG_INFO_BTF=n 关闭此选项

​      (2) 编译完成，但运行新kernel时报错`out of memory`
解决办法：裁剪module大小，编译模块时使用 `make  INSTALL_MOD_STRIP=1 modules_install`，.ko被编译时会缩减非必要的debug信息。

### 2.2 合并UHSII patch后再编译整个Kernel

官方kernel源码可以到[kernel.org](https://www.kernel.org/)下载

合并UHSII patch，仅涉及到mmc模块的代码，如果差异不大可以将linux-uhs2-gl9755-v3-patch的drivers/mmc和include头文件直接拷到待编译kernel的drivers/mmc和include/linux/mmc。

如果是手动合并UHS-II patch，需要考虑以下部分：
- 源码，包括drivers/mmc和include/linux/mmc
- Makefile, 包括drivers/mmc/core和drivers/mmc/host
- Kconfig, 包括drivers/mmc，及其子目录core和host

具体合并方法参考《Linux设备驱动开发详解》
合并完后，Kernel编译流程和上节相同

### 2.3 单独编译MMC模块
一般的驱动开发，都是可以单独编译成module模块，然后用rmmod和insmod替换原系统的模块

但是UHS-II patch涉及到mmc/core层的改动，而core是build-in的，不能作为模块编译，因此只能编译整个kernel。以后如果只修改host层的代码，可以将mmc/host单独编译为module后安装。

待编译kernel目录是`~/linux-5.8-rc4`
```
  #编译模块，"M="指定待编译源码，编译完拷贝.ko到"-C"指定的目录，此目录为系统存放模块的目录
  sudo make -C /lib/modules/`uname -r`/build M=~/linux-5.8-rc4/drivers/mmc
  
  #安装模块
  make -C /lib/modules/`uname -r`/build M=~/linux-5.8-rc4/drivers/mmc modules_install
	
  #清除模块,包括.o和.ko文件
  sudo make -C /lib/modules/`uname -r`/build M=~/linux-5.8-rc4/drivers/mmc clean
```
参考：[# Building External Modules](https://www.kernel.org/doc/html/latest/kbuild/modules.html)

注意，`make xxx modules_install`是不能让模块自动加载的，只是安装到了/lib/modules位置。使用`modinfo`查看模块信息，似乎是使用了/lib/modules下的，但没有实际加载和生效。
要加载模块，两种方法：
1. rmmod/insmod 手动替换, 参考下一节
2. make modules_install 之后再 make install，更新整个kernel, 此后外部模块才会被内核自动加载（通常使用这种方式）

### 2.4 手动替换MMC模块

#### 2.4.1 UHS-II相关模块的依赖关系
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301640662.png)

可以从mmc/host的Kconfig得知依赖：
```
config MMC_SDHCI_PCI
	tristate "SDHCI support on PCI bus"
	depends on MMC_SDHCI && PCI
	select MMC_SDHCI_UHS2
	
config MMC_SDHCI_UHS2
	tristate "UHS2 support on SDHCI controller"
	depends on MMC_SDHCI
```

使用`lsmod`可以得知module依赖关系，如下图，sdhci_uhs2被sdhci_pci引用1次, sdhci被sdhci_uhs2和sdhci_pci引用2次
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301645712.png)
`modinfo`可以得知已加载module的.ko路径
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203301645737.png)

#### 2.4.2 手动卸载和装载module
卸载和装载都要按依赖顺序处理，shell脚本如下.
```
#!/bin/bash

sudo rmmod sdhci_pci
sudo rmmod sdhci_uhs2
sudo rmmod sdhci

sudo insmod /lib/modules/`uname -r`/build/drivers/mmc/host/sdhci.ko
sudo insmod /lib/modules/`uname -r`/build/drivers/mmc/host/sdhci-uhs2.ko
sudo insmod /lib/modules/`uname -r`/build/drivers/mmc/host/sdhci-pci.ko 
```

## 3. 调试过程
### 3.1 调试工具
1. printk
printk是很常用的driver调试手段，配合dmesg查看kernel log可以定位常见问题。
printk如何开启不同打印级别，参考[# Message logging with printk](https://www.kernel.org/doc/html/latest/core-api/printk-basics.html)

例如，使用`dmesg -n 6`开启KERN_INFO级别，然后在driver中添加pr_info()作为info打印, 在dmesg中查看打印log。

注意KERN_DEBUG比较特殊，不仅要`dmesg -n 7`开启, 还需要在driver module的makefile添加Debug CFLAGS, 有两种方法：
```
#该Makefile相关模块全部启用debug
EXTRA_CFLAGS += -DDEBUG

#指定模块启用debug
CFLAGS-xxx-mmc += -DDEBUG
```
示例：使用`pr_info(“enter %s\n”, __FUNCTION__);` 打印函数调用流程

2. dmesg
示例参考 [# How to use the dmesg Command on Linux](https://www.geeksforgeeks.org/how-to-use-the-dmesg-command-on-linux/) 
比较常用的有：
```
sudo dmesg
sudo dmesg -c 
sudo dmesg | head -100
sudo dmesg | tail
sudo dmesg | xxx
```

3.vscode
vscode比vim/gedit更方便直接改代码，用.deb安装容易失败，推荐命令行安装方式：
```
#更新相关microsoft源
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg

sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/

sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/trusted.gpg.d/packages.microsoft.gpg] \
https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

rm -f packages.microsoft.gpg

#安装
sudo apt update
sudo apt install code
```
### 3.2 UHSII调试
1. 模块加载初始化过程中dmesg显示直接dump
   基本是空指针问题，例如：
   - 只编译UHSII host 模块，而不编译kernel的core层，insmod host模块时就会dump, 因为core层相关API不存在。
   - 获取相关数据结构方法不对导致空指针
   例如获取slot要使用：
   ```
   struct sdhci_pci_slot *slot = sdhci_priv(host);
	
	static inline void *sdhci_priv(struct sdhci_host *host){
    return host->private;
    }
   
   ```
   而host->private实际指向sdhci_host结构体的最后定义的如下0长度数组
   ```
   unsigned long private[] ____cacheline_aligned;
   ```
   参考：[# [Explanation on private variable in c struct](https://stackoverflow.com/questions/17931491/explanation-on-private-variable-in-c-struct)](https://stackoverflow.com/questions/17931491/explanation-on-private-variable-in-c-struct)
   基本含义是可以获取结构体外部的数据，而host指针本身确实属于slot结构体sdhci_pci_slot的一部分，所以host->private能访问到slot。

3. 贴一段dmesg log，包含UHSII初始化过程直到最后一步GO_DORMANT fail
具体流程参考UHSII spec:  SD_Specifications_Part_1_UHS_II_Addendum
```
[  522.171631] sdhci_uhs2 [sdhci_uhs2_do_detect_init()]: sdhci_uhs2_do_detect_init: begin UHS2 init.
[  522.171632] enter sdhci_pci_o2_pre_detect_init.
[  522.171632] exit sdhci_pci_o2_pre_detect_init.
[  522.171835] sdhci_uhs2 [sdhci_uhs2_interface_detect()]: mmc0: UHS2 Lane synchronized in UHS2 mode, PHY is initialized.
[  522.171855] uhs2_cmd_assemble: uhs2_cmd: header=0x80 arg=0x292
[  522.171856] uhs2_cmd_assemble:           payload_len=4 packet_len=8 resp_len=6
[  522.171856] [uhs2_dev_init()]: Begin DEVICE_INIT, header=0x80, arg=0x292, payload=0x8808.
[  522.171856] [uhs2_dev_init()]: Sending DEVICE_INIT. Count = 0
[  522.171858] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.171865] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x800 is set to UHS2 CMD register.
[  522.171874] mmc0: sdhci: IRQ status 0x00000001
[  522.171885] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.171887] uhs2_cmd_assemble: uhs2_cmd: header=0x80 arg=0x292
[  522.171887] uhs2_cmd_assemble:           payload_len=4 packet_len=8 resp_len=6
[  522.171888] [uhs2_dev_init()]: Begin DEVICE_INIT, header=0x80, arg=0x292, payload=0x8808.
[  522.171888] [uhs2_dev_init()]: Sending DEVICE_INIT. Count = 1
[  522.171888] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.171894] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x800 is set to UHS2 CMD register.
[  522.188184] mmc0: sdhci: IRQ status 0x00000001
[  522.188205] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188254] [uhs2_dev_init()]: CF is set, device is initialized!
[  522.188257] [uhs2_enum()]: Begin ENUMERATE, header=0x80, arg=0x392, payload=0xf0.
[  522.188260] uhs2_cmd_assemble: uhs2_cmd: header=0x80 arg=0x392
[  522.188262] uhs2_cmd_assemble:           payload_len=4 packet_len=8 resp_len=8
[  522.188266] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188277] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x800 is set to UHS2 CMD register.
[  522.188290] mmc0: sdhci: IRQ status 0x00000001
[  522.188308] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188318] [uhs2_enum()]: id_f = 6, id_l = 6.
[  522.188320] [uhs2_enum()]: Enumerate Cmd Completed. No. of Devices connected = 1
[  522.188322] [uhs2_config_read()]: INQUIRY_CFG: read Generic Caps.
[  522.188324] [uhs2_config_read()]: Begin INQUIRY_CFG, header=0x86, arg=0x10.
[  522.188326] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x10
[  522.188328] uhs2_cmd_assemble:           payload_len=0 packet_len=4 resp_len=0
[  522.188331] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188342] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x400 is set to UHS2 CMD register.
[  522.188363] mmc0: sdhci: IRQ status 0x00000001
[  522.188392] mmc0: req done (CMD0): 0: 00010100 00000000 00000000 00000000
[  522.188398] [uhs2_config_read()]: Device Generic Caps (0-31) is: 0x10100.
[  522.188399] [uhs2_config_read()]: INQUIRY_CFG: read PHY Caps.
[  522.188401] [uhs2_config_read()]: Begin INQUIRY_CFG, header=0x86, arg=0x220.
[  522.188404] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x220
[  522.188410] uhs2_cmd_assemble:           payload_len=0 packet_len=4 resp_len=0
[  522.188415] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188427] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x400 is set to UHS2 CMD register.
[  522.188447] mmc0: sdhci: IRQ status 0x00000001
[  522.188476] mmc0: req done (CMD0): 0: 00008000 00000080 00000000 00000000
[  522.188482] [uhs2_config_read()]: Device PHY Caps (0-31) is: 0x8000.
[  522.188484] [uhs2_config_read()]: Device PHY Caps (32-63) is: 0x80.
[  522.188487] [uhs2_config_read()]: INQUIRY_CFG: read LINK-TRAN Caps.
[  522.188492] [uhs2_config_read()]: Begin INQUIRY_CFG, header=0x86, arg=0x420.
[  522.188499] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x420
[  522.188504] uhs2_cmd_assemble:           payload_len=0 packet_len=4 resp_len=0
[  522.188507] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188516] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x400 is set to UHS2 CMD register.
[  522.188554] mmc0: sdhci: IRQ status 0x00000001
[  522.188582] mmc0: req done (CMD0): 0: 20024000 00000000 00000000 00000000
[  522.188601] [uhs2_config_read()]: Device LINK-TRAN Caps (0-31) is: 0x20024000.
[  522.188604] [uhs2_config_read()]: Device LINK-TRAN Caps (32-63) is: 0x0.
[  522.188605] [uhs2_config_write()]: SET_COMMON_CFG: write Generic Settings.
[  522.188607] [uhs2_config_write()]: Both Host and device support 2L-HD.
[  522.188609] [uhs2_config_write()]: Begin SET_COMMON_CFG, header=0x86, arg=0x8a0
[  522.188611] [uhs2_config_write()]: UHS2 write Generic Settings 00000000 00000000
[  522.188613] [uhs2_config_write()]: flags=00000005 dev_prop.n_lanes_set=0 host_caps.n_lanes_set=0
[  522.188615] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x8a0
[  522.188618] uhs2_cmd_assemble:           payload_len=8 packet_len=12 resp_len=0
[  522.188620] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188632] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0xc00 is set to UHS2 CMD register.
[  522.188650] mmc0: sdhci: IRQ status 0x00000001
[  522.188678] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188684] [uhs2_config_write()]: SET_COMMON_CFG: PHY Settings.
[  522.188686] [uhs2_config_write()]: set dev_prop.speed_range_set to SPEED_B
[  522.188689] [uhs2_config_write()]: UHS2 SET PHY Settings  40000000 04000000
[  522.188691] [uhs2_config_write()]: host->flags=00000015 dev_prop.speed_range_set=1
[  522.188693] [uhs2_config_write()]: dev_prop.n_lss_sync_set=4 host_caps.n_lss_sync_set=4
[  522.188694] [uhs2_config_write()]: dev_prop.n_lss_dir_set=0 host_caps.n_lss_dir_set=8
[  522.188696] [uhs2_config_write()]: Begin SET_COMMON_CFG header=0x86 arg=0xaa0
[  522.188698] [uhs2_config_write()]: 		payload[0]=0x40000000 payload[1]=0x4000000
[  522.188700] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0xaa0
[  522.188703] uhs2_cmd_assemble:           payload_len=8 packet_len=12 resp_len=4
[  522.188705] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188715] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0xc00 is set to UHS2 CMD register.
[  522.188730] mmc0: sdhci: IRQ status 0x00000001
[  522.188741] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188746] [uhs2_config_write()]: SET_COMMON_CFG: LINK-TRAN Settings.
[  522.188748] [uhs2_config_write()]: Begin SET_COMMON_CFG header=0x86 arg=0xca0
[  522.188750] [uhs2_config_write()]: 		payload[0]=0x80320 payload[1]=0x1000000
[  522.188752] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0xca0
[  522.188754] uhs2_cmd_assemble:           payload_len=8 packet_len=12 resp_len=0
[  522.188756] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188766] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0xc00 is set to UHS2 CMD register.
[  522.188780] mmc0: sdhci: IRQ status 0x00000001
[  522.188808] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188813] [uhs2_config_write()]: SET_COMMON_CFG: Set Config Completion.
[  522.188815] [uhs2_config_write()]: Begin SET_COMMON_CFG, header=0x86, arg=0x8a0, payload[0] = 0x0.
[  522.188817] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x8a0
[  522.188819] uhs2_cmd_assemble:           payload_len=8 packet_len=12 resp_len=5
[  522.188821] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.188831] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0xc00 is set to UHS2 CMD register.
[  522.188842] mmc0: sdhci: IRQ status 0x00000001
[  522.188855] mmc0: req done (CMD0): 0: 00000000 00000000 00000000 00000000
[  522.188862] sdhci_uhs2 [sdhci_uhs2_do_set_reg()]: Begin sdhci_uhs2_set_reg, act 0.
[  522.201612] sdhci_uhs2 [sdhci_uhs2_do_set_reg()]: Begin sdhci_uhs2_set_reg, act 3.
[  522.201614] sdhci_uhs2 [sdhci_uhs2_do_set_reg()]: Begin sdhci_uhs2_set_reg, act 2.
[  522.201616] [uhs2_go_dormant()]: Begin GO_DORMANT_STATE, header=0x86, arg=0x192, payload=0x0.
[  522.201617] uhs2_cmd_assemble: uhs2_cmd: header=0x86 arg=0x192
[  522.201618] uhs2_cmd_assemble:           payload_len=4 packet_len=8 resp_len=0
[  522.201619] mmc0: starting CMD0 arg 00000000 flags 00000000
[  522.201626] sdhci_uhs2 [sdhci_uhs2_send_command()]: 0x8c0 is set to UHS2 CMD register.
[  522.218633] mmc0: sdhci: IRQ status 0x00008000
[  522.218636] sdhci_uhs2 [sdhci_uhs2_irq()]: *** mmc0 got UHS2 interrupt: 0x00010000
[  522.218651] enter sdhci_pci_o2_reset.
[  522.218652] enter o2_uhs2_reset_sd_tran.
[  522.218652] exit o2_uhs2_reset_sd_tran.
[  522.218654] enter sdhci_pci_o2_reset.
[  522.218654] enter o2_uhs2_reset_sd_tran.
[  522.218654] exit o2_uhs2_reset_sd_tran.
[  522.218659] mmc0: req done (CMD0): -110: 00000000 00000000 00000000 00000000
[  522.218666] mmc0: uhs2_go_dormant: UHS2 CMD send fail, err= 0xffffff92!
[  522.218668] mmc0: uhs2_change_speed: UHS2 GO_DORMANT_STATE fail, err= 0xfffffffb!
[  522.218669] mmc0: UHS2 uhs2_change_speed() fail!
```
含义是UHSII初始化接近完成，切换到高速的RangeB时，GO_DORMANT_STATE命令未完成，超时。
解决办法：先绕过RangeB模式，使用RangA(较低速度的UHSII模式)，为此要从一开始就上报host不支持RangeB。
修改mmc/host/sdhci-uhs2.c中的上报host能力(capability)的speed_range为不支持RangeB
```
//mmc->uhs2_caps.speed_range =(caps_phy & SDHCI_UHS2_HOST_CAPS_PHY_RANGE_MASK) >> SDHCI_UHS2_HOST_CAPS_PHY_RANGE_SHIFT;

mmc->uhs2_caps.speed_range = 0; //Range-A
```
重新编译安装module后，UHSII初始化正常，读写正常。

事实上此GO_DORMANT fail issue的根本原因是兼容性问题：
UHSII初始化流程中，SD host侧对lane speed的配置最好在卡处在dormant状态下进行，host侧提高速度（从Range-A提高到RangeB）以后，卡侧在退出dormant状态时重新配置速度，和host速度匹配。
如果host侧修改lane speed时间点错误，有的SD卡来不及反应，不能同步速度，所以GO_DORMANT fail；而有的SD 卡性能好，随时同步host侧的速度，没有此issue。

另外有的Issue和硬件特性相关，例如上电需要等待一定时间以后，才能启动UHSII设备初始化，这个等待时间取决于SD host厂商的硬件特性。