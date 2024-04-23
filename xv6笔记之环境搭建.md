---
title: xv6笔记之环境搭建
date: 2023-05-06 17:37:22
tags: xv6
categories: xv6
---
本文在Ubuntu22.04上搭建xv6(x86版本)的开发环境，用于编译、调试xv6源码。

- xv6 x86版本参考[MIT6.828/2018](https://pdos.csail.mit.edu/6.828/2018/overview.html)
- xv6 riscv版本参考[MIT6.S081](https://pdos.csail.mit.edu/6.828/2020/) ，MIT6.828从2019年以后以RISCV指令集实现，并拆分了课程

两者的课程内容区别：

6.828 and 6.S081 will be offered as two separate classes. 6.S081 (Introduction to Operating Systems) will be taught as a stand-alone AUS subject for undergraduates, and will provide an introduction to operating systems. 6.828 will be offered as a graduate-level seminar-style class focused on research in operating systems. 6.828 will assume you have taken 6.S081 or an equivalent class.

为什么选用x86版本：

x86版本有更完善的资料和更细节的代码讲解，参考：

[xv6-annotated](https://github.com/palladian1/xv6-annotated)

[woai3c/MIT6.828](https://github.com/woai3c/MIT6.828)

学完x86版本再学riscv版本，只需要关注指令集差异即可

## 编译工具链

主流程参考：[Tools Used in 6.828](https://pdos.csail.mit.edu/6.828/2018/tools.html)

这里只记录我操作过程中和该wiki的差异点

1.下载包有的连接失败，bing搜索到合适的下载源后，最终成功下载的操作如下：

```
wget https://www.mirrorservice.org/pub/gnu/gmp/gmp-5.0.2.tar.bz2
wget https://www.mpfr.org/mpfr-3.1.2/mpfr-3.1.2.tar.bz2
wget http://www.multiprecision.org/downloads/mpc-0.9.tar.gz
wget http://ftpmirror.gnu.org/binutils/binutils-2.21.1.tar.bz2
wget http://ftpmirror.gnu.org/gcc/gcc-4.6.4/gcc-core-4.6.4.tar.bz2
wget http://ftpmirror.gnu.org/gdb/gdb-7.3.1.tar.bz2
```

2.编译Toolchain中的问题：

(0)**注意!!!** 在编译Toolchain完成以后要恢复默认的LD_LIBRARY_PATH，不要在toolchain配置了LD_LIBRARY_PATH的情况下去完成后续的安装qemu等其他任何操作，否则可能系统损害无法进入桌面且不能recovery，报错如下:

```
libgnutls.so.30 undefined symbol: __gmpz_limbs_write
```

问题原因和解决办法参考：[[apt-get wants an older GNUTLS version to be defined](https://askubuntu.com/questions/1065651/apt-get-wants-an-older-gnutls-version-to-be-defined)](https://askubuntu.com/questions/1065651/apt-get-wants-an-older-gnutls-version-to-be-defined)

**LIB PATH导致系统损坏的经验：搭建开发环境配置的LD_LIBRARY_PATH不要随便export；在使用时export, 使用完毕后恢复**

(1)如果安装在/usr/local，所有make install都要sudo；安装在home不需要sudo

(2)编译gcc时报错：`configure: error: cannot compute suffix of object files: cannot compile`

需要export PATH，由于所有编译包都安装在/usr/local/，所以export PATH也为/usr/local/，保存为export-path.sh方便重启后使用，也可以加到~/.bashrc：

```
export PATH=$PFX/bin:$PATH
export LD_LIBRARY_PATH=$PFX/lib:$LD_LIBRARY_PATH
```

(3)编译gdb时报错：`error: no termcap library found`

要手动下载termcap包并编译，操作过程和toolchain一样：

```
wget https://ftp.gnu.org/gnu/termcap/termcap-1.3.1.tar.gz
tar xzf termcap-1.3.1.tar.gz
cd termcap-1.3.1/
./configure --prefix=$PFX
make && sudo make install
cd ..
```

完整的编译脚本:

```
#!/bin/bash

export PFX=~/xv6/toolchain #这里编译到home,也可以用/usr/local
mkdir -p $PFX
cd $PFX

#install a development environment.
sudo apt-get install -y build-essential gdb
sudo apt-get install gcc-multilib

#Building Your Own Compiler Toolchain
#wget容易失败，因此这部分最好手动执行，确保全部下载成功
wget https://www.mirrorservice.org/pub/gnu/gmp/gmp-5.0.2.tar.bz2
wget https://www.mpfr.org/mpfr-3.1.2/mpfr-3.1.2.tar.bz2
wget http://www.multiprecision.org/downloads/mpc-0.9.tar.gz
wget http://ftpmirror.gnu.org/binutils/binutils-2.21.1.tar.bz2
wget http://ftpmirror.gnu.org/gcc/gcc-4.6.4/gcc-core-4.6.4.tar.bz2
wget http://ftpmirror.gnu.org/gdb/gdb-7.3.1.tar.bz2

export PATH=$PFX/bin:$PATH
export LD_LIBRARY_PATH=$PFX/lib:$LD_LIBRARY_PATH

tar xjf gmp-5.0.2.tar.bz2
cd gmp-5.0.2
./configure --prefix=$PFX
make
make install             # This step may require privilege (sudo make install)
cd ..

tar xjf mpfr-3.1.2.tar.bz2
cd mpfr-3.1.2
./configure --prefix=$PFX --with-gmp=$PFX #这里指定gmp.h的path
make
make install             # This step may require privilege (sudo make install)
cd ..

tar xzf mpc-0.9.tar.gz
cd mpc-0.9
./configure --prefix=$PFX --with-gmp=$PFX #这里指定gmp.h的path
make
make install             # This step may require privilege (sudo make install)
cd ..

tar xjf binutils-2.21.1.tar.bz2
cd binutils-2.21.1
./configure --prefix=$PFX --target=i386-jos-elf --disable-werror
make
make install             # This step may require privilege (sudo make install)
cd ..

tar xjf gcc-core-4.6.4.tar.bz2
cd gcc-4.6.4
mkdir build              # GCC will not compile correctly unless you build in a separate directory
cd build
../configure --prefix=$PFX \ 
    --with-gmp=$PFX --with-mpfr=$PFX --with-mpc=$PFX \ #指定gmp, mpfr, mpc位置
    --target=i386-jos-elf --disable-werror \
    --disable-libssp --disable-libmudflap --with-newlib \
    --without-headers --enable-languages=c MAKEINFO=missing
make all-gcc
make install-gcc         # This step may require privilege (sudo make install-gcc)
make all-target-libgcc
make install-target-libgcc     # This step may require privilege (sudo make install-target-libgcc)
cd ../..

wget https://ftp.gnu.org/gnu/termcap/termcap-1.3.1.tar.gz
tar xzf termcap-1.3.1.tar.gz
cd termcap-1.3.1/
./configure --prefix=$PFX
make && make install
cd ..

tar xjf gdb-7.3.1.tar.bz2
cd gdb-7.3.1
./configure --prefix=$PFX \
    --target=i386-jos-elf --program-prefix=i386-jos-elf- \
    --disable-werror
make all
make install             # This step may require privilege (sudo make install)
cd ..

i386-jos-elf-objdump -i
# Should produce output like:
# BFD header file version (GNU Binutils) 2.21.1
# elf32-i386
#  (header little endian, data little endian)
#   i386...

i386-jos-elf-gcc -v
# Should produce output like:
# Using built-in specs.
# COLLECT_GCC=i386-jos-elf-gcc
# COLLECT_LTO_WRAPPER=/usr/local/libexec/gcc/i386-jos-elf/4.6.4/lto-wrapper
# Target: i386-jos-elf

export LD_LIBRARY_PATH="" #恢复系统本身的libpath(默认空)，避免装其他软件有lib冲突造成系统损坏
```

## 交叉编译的参数

在交叉编译configure时，通常会需要设置--build、--host和--target选项。各个选项的含义如下：

- --build：编译所用的机器的平台。
- --host：编译出的代码运行的平台。
- --target：编译出来的工具链生成的代码的运行平台。这个选项不常用，一般只在编译gcc、ld等工具链的过程中用到。

在不涉及到交叉编译的时候，--build、--host、--target缺省值都是本机平台，不需要特别设置。

在交叉编译的时候，比如需要在x86平台编译arm程序，就需要设置--build和--host选项；其中host的内容为目标平台名称，通常编译器的名字前缀就是目标平台名称，例如用arm-unknown-linux-gnueabi-gcc编译，--host设置为arm-unknown-linux-gnueabi；--build可以缺省不设置就是使用当前平台名称

## 编译QEMU

xv6使用的QEMU是patched version，要手动编译，过程如下：

```
git clone https://github.com/mit-pdos/6.828-qemu.git qemu

sudo apt install libsdl1.2-dev libtool-bin libglib2.0-dev libz-dev libpixman-1-dev

#此qemu版本需要python2 (2.7), 由于python2和3不兼容, 且系统只有Python3, 因此需要安装
sudo apt install python2
python2 -V
cd qemu

./configure --disable-kvm --disable-werror --prefix=$PFX --target-list="i386-softmmu x86_64-softmmu" --python=/usr/bin/python2

make && make install
```

qemu编译错误的解决办法：[MIT6.828 实验环境安装教程](https://github.com/woai3c/MIT6.828/blob/master/docs/install.md)

其中以下错误的解决方法： 在 `qga/commands-posix.c` 文件中加 `#include <sys/sysmacros.h>`

```
qga/commands-posix.c: In function ‘dev_major_minor’:
qga/commands-posix.c:633:13: error: In the GNU C Library, "major" is defined
 by <sys/sysmacros.h>.
```

## 运行xv6

下载6.828的jos lab，make产生kernel.img

```
git clone https://pdos.csail.mit.edu/6.828/2018/jos.git lab
```

运行qemu的xv6之前，需要export PATH和LD_LIBRARY_PATH；运行之后要清掉LD_LIBRARY_PATH为空(重启或手动清除)

写export_xv6.sh如下:

```
export PFX=~/xv6/toolchain
export PATH=$PFX/bin:$PATH
export LD_LIBRARY_PATH=$PFX/lib:$LD_LIBRARY_PATH
```

建议给调用export_xv6.sh的命令加别名(alias)到.bashrc，可以用get-xv6命令一键export：

```
alias get_xv6='. $HOME/xv6/export_xv6.sh'
```

如果是本地执行qemu(带GUI)用`make qemu`; 如果是远程终端执行用`make qemu-nox`。qemu内容如下表示qemu环境搭建OK

```
sed "s/localhost:1234/localhost:26000/" < .gdbinit.tmpl > .gdbinit
qemu-system-i386 -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26000 -D qemu.log 
VNC server running on `127.0.0.1:5900'
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> 
```

