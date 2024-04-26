# 串口调试环境

## 串口硬件连接

串口线的颜色和功能是对应的，4根线如下：

```
功能    	   线的颜色
TX           绿色
RX           白色
GND          黑色
VCC          红色
```

串口的接口类型取决于开发板接口，飞腾Pi是使用USB转TTL接口，有的开发板是板载串口转USB芯片，外部是USB直连PC，还有的是RS232接口。不管接口类型如何，串口转接芯片一般是CH340/FT232。

![image-20240424142615021](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404241426062.png)

飞腾派接线只用了3根线，没用VCC。注意RX是接到TX， TX是接到RX。

![image-20240424142831480](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404241428560.png)

验证串口连接正常：查找到ttyUSB设备即USB转串口设备连接正常。

```
ls /dev/ttyUSB0
```

注：Ubuntu有时串口线找不到ttyUSB0设备，可能跟线的质量或者芯片驱动相关。在Ubuntu下FT232不需要安装驱动能稳定连接。Windows下串口转接芯片一定需要装对应的CH340或FT232驱动才能在设备管理器COM口看到USB设备。

## Putty 配置串口调试环境

Putty连接串口需要以root用户启动。非root用户启动putty无法设置端口号，波特率等参数。

以root启动putty有两种方式：

- 以root启动putty GUI，在GUI中配置端口号，波特率等参数。
- 以命令行sudo启动putty，在启动参数中配置端口号，波特率等参数。

以root启动putty GUI：对于Ubuntu 22.04 Wayland桌面，使用`sudo -E program`启动GUI。

参考：https://wiki.archlinux.org/title/Running_GUI_applications_as_root

以命令行sudo启动putty：

参考：https://readypinaple.com/using-putty-serial-terminal

对于我的调试环境（飞腾派），参数如下：

`sudo putty -serial /dev/ttyUSB0 -sercfg 115200,8,n,1,N`

## Putty串口环境的复制和粘贴

Ubuntu22.04环境上有两种方法：

1.使用鼠标中键在terminal和putty之间复制和粘贴；此方法不能在选中之后再选中，否则内容被覆盖；

2.启动putty GUI后在selection设置ctrl+shift +C/V 复制粘贴。记得保存设置到session的default配置。terminal默认支持ctrl+shift+C/V操作。这种方式更稳定。

以上的ctrl+shift+C/V和中键操作属于两套剪贴板buffer，不会相互覆盖。即ctrl+shift+C复制的数据必须用ctrl+shift+V粘贴，中键选中的数据必须用中键粘贴。

## Putty记录和查看log

### 记录log

在启动GUI后在session->logging设置记录all output，记得保存设置到session的default配置。

### 查看log

查看log可以使用less显示 + grep过滤；

**关于less：** 类似vim快捷键.

匹配查找：在less中查找可以使用 /或者？查找，参考以下less的help：

```
/pattern          *  Search forward for (N-th) matching line.
?pattern          *  Search backward for (N-th) matching line.
n                 *  Repeat previous search (for N-th occurrence).
N                 *  Repeat previous search in reverse direction.

#对于有空格的匹配查找，用反斜杠转义空格，例如：
/this\ is\ pattern
```

跳转到头尾：大G（跳到末尾）和 gg（重来到开头）.

**关于grep**，有几点很常用：

a.有时log中有乱码导致被grep设别为binary导致grep无法显示，需要`grep --text`参数指定以文本打开：

例如要查看putty.log中mmc0模块的打印：

```
cursorhu@ubuntu-PC:~/phytium/phytium-linux-kernel-v1.0.1$ less ~/putty.log | grep mmc0 --text

[    2.208568] mmc0: mmc_sd_setup_card:
[    2.208571] mmc0: phytium_mci_ops_request: cmd:55 arg:0x400000
[    2.209331] mmc0: phytium_mci_irq: events:100,mask:0x1547,dmac_events:0,dmac_mask:0x0,cmd:55
[    2.209335] mmc0: phytium_mci_err_irq:
[    2.209348] mmc0: error -110 whilst initialising SD card
```

b.有时需要grep多个关键字，使用`grep -E “A|B”`，注意一定要用引号扩起来

例如要同时查看mmc0和BHT的打印：

```
less ~/putty.log | grep -E "mmc0|BHT" --text

[    6.816021] BHT MSG:sdr50_notuning_sela_rx_inject:463
[    6.821067] BHT MSG:exit:_ggc_output_tuning  0
[    6.825505] BHT MSG: finit is 400000Hz
[    6.829248] mmc0: tuning execution failed: 1
[    6.833513] mmc0: phytium_mci_ops_set_ios:
[    6.841521] mmc0: phytium_mci_set_buswidth: width = 0, set value:0x0
```

c.有时需要排除某关键字，例如飞腾的SD host有mmc0, mmc1两个，我只关注mmc0, 使用 grep -v KEYWORD去排除， v：invert match

```
less ~/putty.log | grep --text -v mmc1
```

**less+grep+less：**

用less+grep过滤出log另存为新log， 再用less查看

```
less ~/putty.log | grep --text -E "mmc0|BHT" > putty-bh201-tuning-error.log 
less putty-bh201-tuning-error2.log
```

# 交叉编译

## 交叉编译方法

设置工具链环境变量+交叉编译命令如下：

```
set_env.sh:

export PATH=$PATH:"/opt/toolchains/gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu/bin"
export CROSS_COMPILE=aarch64-linux-gnu-
export ARCH=arm64

build_kernel.sh:

make ARCH=arm64 e2000_defconfig all -j4 INSTALL_MOD_PATH=modules modules_install
```

安装编译的系统镜像和设备树：把编译机的输出image和dtb拷贝到SD卡系统盘的rootfs/boot/里，

```
cp_image.sh：

sudo cp arch/arm64/boot/Image /media/cursorhu/rootfs/boot/
sudo cp arch/arm64/boot/dts/phytium/phytium-pi-board.dtb /media/cursorhu/rootfs/boot/
```

SD卡放回开发板启动，putty串口回车进入uboot，指定从SD卡系统盘的rootfs/boot/启动内核

```
uboot.sh：（并不能直接在uboot运行）

setenv bootargs console=ttyAMA1,115200 earlycon=pl011,0x2800d000 root=/dev/mmcblk0p1 rootwait rw;
ext4load mmc 0:1 0x90100000 boot/Image;
ext4load mmc 0:1 0x90000000 boot/phytium-pi-board.dtb;
booti 0x90100000 – 0x90000000;
```

# uboot配置

## uboot自动选择指定kernel启动

背景：如下图，飞腾派启动指定内核需要设置uboot参数，由于飞腾派的uboot源码没有开放，不能用脚本（boot.scr）。调试新编译的内核，每次飞腾派启动都需要在putty粘贴uboot命令。开发效率低。

![Screenshot from 2024-04-08 15-40-55](/home/cursorhu/Pictures/Screenshot from 2024-04-08 15-40-55.png)

解决方案：将uboot命令配置到uboot环境变量，并保存到系统盘中使断电重启后uboot环境变量也生效。

参考mastering embeded linux programming， 使用bootcmd参数可以配置uboot script去执行一连串命令，实现类似shell script功能并保存，每次启动uboot不用复制粘贴一条条命令去启动。 

以飞腾派为例，每次编译kernel后要从新编译的kernel启动，需要每次手动设置以下uboot命令/环境：

```
#启动参数环境变量
setenv bootargs console=ttyAMA1,115200 earlycon=pl011,0x2800d000 root=/dev/mmcblk0p1 rootwait rw;
#命令语句（非环境变量）
ext4load mmc 0:1 0x90100000 boot/Image;
ext4load mmc 0:1 0x90000000 boot/phytium-pi-board.dtb;
booti 0x90100000 – 0x90000000;
```

目标：使uboot自动从以上命令语句指定的kernel启动：

对于环境变量bootargs不需要改动，我们只需要将命令语句设置到bootcmd环境变量。最后再一起saveenv使两个环境变量都保存到硬盘，重启后能自动生效即可。

1. 如何设置bootcmd到uboot环境变量：

将以上uboot命令语句合成为一条语句设置到bootcmd：uboot命令的结束标志是分号，设置bootcmd时在内容语句的每个分号前需要加转移字符 \, 这样setenv才会判断这个分号是要输入到bootcmd的字符，而不散表明setenv命令本身的结束。 

```
setenv bootcmd ext4load mmc 0:1 0x90100000 boot/Image\;ext4load mmc 0:1 0x90000000 boot/phytium-pi-board.dtb\;booti 0x90100000 – 0x90000000\;
```

注意：上面指令拷贝到putty串口时时符号 `-`会消失，需要手动添加

2. 如何使bootargs和bootcmd环境变量重启后也自动生效：保存（所有）环境变量

```
saveenv
```

3. 重启后可以printenv查看配置生效，根据kernel打印可确认确实从指定kernel启动。

```
Phytium-Pi#printenv
arch=arm
baudrate=115200
board=e2000
board_name=e2000
boot_os=bootm $kernel_addr -:- $ft_fdt_addr
bootargs=console=ttyAMA1,115200 earlycon=pl011,0x2800d000 root=/dev/mmcblk0p1 rootwait rw;
bootcmd=ext4load mmc 0:1 0x90100000 boot/Image;ext4load mmc 0:1 0x90000000 boot/phytium-pi-board.dtb;booti 0x90100000 - 0x90000000;
bootdelay=2
cpu=armv8
```

5.如何删除env：变量设置为空即为删除：

```
setenv bootargs
setenv bootcmd
saveenv
```

# 附录：问题记录

## ARM工具链报错问题

编译飞腾kernel有时会报错：

```
cursorhu@ubuntu-PC:~/phytium/phytium-linux-kernel-v1.0.1$ ./build_kernel.sh 
#
# configuration written to .config
#
arch/arm64/Makefile:27: ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum
arch/arm64/Makefile:40: LSE atomics not supported by binutils
arch/arm64/Makefile:48: Detected assembler with broken .inst; disassembly will be unreliable
scripts/kconfig/conf  --syncconfig Kconfig
arch/arm64/Makefile:27: ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum
arch/arm64/Makefile:40: LSE atomics not supported by binutils
arch/arm64/Makefile:48: Detected assembler with broken .inst; disassembly will be unreliable
  CC      scripts/mod/empty.o
gcc: error: unrecognized command-line option ‘-mlittle-endian’
make[3]: *** [scripts/Makefile.build:304: scripts/mod/empty.o] Error 1
make[2]: *** [scripts/Makefile.build:544: scripts/mod] Error 2
make[1]: *** [Makefile:1085: scripts] Error 2
make[1]: *** Waiting for unfinished jobs....
make: *** [Makefile:286: __build_one_by_one] Error 2

```

报错内容指向以下patch：

https://patchwork.kernel.org/project/linux-arm-kernel/patch/1471873737-7614-1-git-send-email-will.deacon@arm.com/

解决方法：删除kernel源码目录的.config文件，重新编译就OK了

相关原因：编译的一些配置：ARCH=arm CROSS_COMPILE=XXX XXX_defconfig都被写入到.confg文件，可能是这个缓存文件异常，根本原因可能是这个版本的工具链有issue。

