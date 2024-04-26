---
title: Ubuntu使用笔记
date: 2023-04-26 11:08:09
tags: ubuntu
categories: linux
---

本文基于Ubuntu 22.04 LTS

## 软件下载源

使用国内软件源下载：

```
software&updates -> Ubuntu Software -> download from -> cn99.com或aliyun.com
```

## 中文输入法

安装中文输入法(pinyin)的步骤：

安装中文支持：

```
Settings -> Region&language -> Manage Installed Languages -> Install/Remove Languages -> 安装chinese simplified
```

设置系统语言为中文：

```
Settings -> Region&language -> Language改成Chinese
```

安装Fcitx框架和中文输入法：

```
sudo apt-get install fcitx-bin #安装fcitx框架
sudo apt-get install fcitx-table #安装输入法栏，其中自动安装拼音输入法
fcitx --version
```

使用Fcitx框架，重启

```
Settings -> Region&language -> Manage Installed Languages -> Keyboard input method system 选择Fcitx 4
```

添加输入法

```
Ubuntu右上角的小键盘图标 -> configure -> 添加pinyin（只有系统语言为中文时才能添加中文输入法）
```

切换中英文输入法：

```
ctrl + space
```

设置系统语言改回英文：

```
Settings -> Region&language -> Language改成English
```

## snap包管理工具

[Snap](https://snapcraft.io/store)是Canonical开发的Linux包管理和软件部署工具。 

安装和使用参考 [**How to Install Snap on Ubuntu**](https://phoenixnap.com/kb/install-snap-ubuntu#:~:text=1%20Start%20by%20updating%20packages%3A%0Asudo%20apt,update%202%20Enter%20the%20following%20command%3A)

特点：丰富的第三方工具库，包括开源工具和闭源工具；二进制安装，不是源码编译

相比apt，其查找工具和安装极为简单：

```
sudo snap find <keyword> #查找keyword相关的工具，显示可安装的列表
sudo snap install <package> #安装列表中的工具
```

查看和卸载snap安装的包：

```
snap list
sudo snap remove <package>
```

示例：安装VSCode和Chrome

```
sudo snap find vscode #找到<package>为code
sudo snap install code --classic
sudo snap find chrome #找到<package>为chromium
sudo snap install chromium
```

## 设置快捷键

setting -> keyboard -> shortcuts -> custom shortcut -> 为应用程序添加快捷键

以截图工具flameshot为例，设置快捷键的command为调用flameshot的命令，截图默认保存到~/Pictures

![image-20230505105544618](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202305051055723.png)

要配置其他flameshot命令的快捷键，用 `man flameshot` 查看，参考 [Keyboard shortcuts for Flameshot](https://flameshot.org/docs/guide/key-bindings/)

## Timeshift备份系统

22.04系统似乎比较容易挂，进不了系统显示"Oh no... system can't recover..."，比如：

Nvdia驱动选择开源版本xserver就挂了一次, recovery模式看/var/log/message有nouveau和nvidia module相关问题

学习xv6时安装编译环境时也挂了一次(不能安装到/usr/local，应该安装到/home)，recovery模式dpkg report显示failure log：

```
symbol lookup error: /lib/x86_64-linux-gnu/libgnutls.so.30: undefined symbol: __gmpz_limbs_write 
```

都是找遍办法都修复不了，只能重装...

为了解决此问题，使用Timeshift将系统备份，参考: [How to Backup and Restore Linux System Settings With Timeshift](https://itsfoss.com/backup-restore-linux-timeshift/)

安装timeshift：

```
sudo apt install timeshift
```

备份整个系统，包括/root和/home/user，设置定时备份

如何恢复：

情景一：系统无法进入桌面，但是可以进入recovery模式root操作：

如下图，用`timeshift --help`查看各种命令，使用`timeshift --restore`恢复指定snapshot

![image-20230508193100794](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202305081931009.png)

情景二：系统无法进入recovery模式，但是备份的snapshot数据还在

使用[Ubuntu Live USB](https://www.ubuntu.com/download/desktop/create-a-usb-stick-on-windows?ref=itsfoss.com) ，即装系统的USB进入try ubuntu环境，联网换国内源安装timeshift，再恢复系统盘中的snapshot数据

情景三：磁盘中的snapshot数据损害：只能重装系统，为了避免此情况发生，应该将系统备份到其他硬盘而不仅仅在当前系统盘

## Clonezilla克隆系统

类似windows ghost的整盘克隆：

https://www.linuxbabe.com/backup/how-to-use-clonezilla-live

至少需要三个盘：

在U盘写入Clonezilla的live usb iso生成Clonezilla live USB，再以Clonezilla live USB启动，对待备份的SSD盘做系统备份，到另一个SSD或者大USB盘；

恢复也是需要Clonezilla live USB + 有系统备份的盘 + 目标写入盘

## 关于系统目录

/usr：系统级的目录，可以理解为C:/Windows/，apt安装的一般在/usr/bin和/usr/lib

/usr/lib：理解为C:/Windows/System32

/usr/local：用户级的程序目录，可以理解为C:/Progrem Files/，用户自己编译的软件默认安装到这个目录下

/opt是用户级的目录用来安装大型的第三方附加软件包，可以理解为D:/Software

开发过程中为了避免lib冲突，自己编译的包建议放在/home/<具体的项目目录>，此外注意自己编译基础库设置的LD_LIBRARY_PATH造成系统库链接冲突

## Tmux

参考：[Tmux 使用教程](https://www.ruanyifeng.com/blog/2019/10/tmux.html)

## VNC远程桌面

Ubuntu安装vino作为VNC server, windows端使用VNC Viewer作为client.

```
apt install vino
setting -> Sharing -> Remote Desktop -> On
```

参考 [Ubuntu 22.04 Remote Desktop Access with Vino](https://www.answertopia.com/ubuntu/ubuntu-remote-desktop-access-with-vino/)
