---
title: Arch Linux安装和配置笔记
date: 2023-07-12 15:50:54
tags: archlinux
categories: linux
---

## 前言

最近Debian12发布，尝鲜在移动硬盘装了Debian12+KDE，但不习惯Debian繁琐的包管理和Ubuntu越来越商业化的行为，最终切换到ArchLinux，进入Pacman和AUR的包管理和滚动更新风格。此文记录ArchLinux安装配置过程。

## 构建多系统的U盘启动盘

使用[ventoy/Start to use Ventoy](https://www.ventoy.net/en/doc_start.html)，将各系统镜像放到Ventoy目录即可，不需要用传统的UltraISO那种一个系统ISO要占用整个U盘。
Arch Linux的ISO[在此下载](https://archlinux.org/download/)

## 使用archinstall安装ArchLinux

[传统的ArchLinux安装方式](https://wiki.archlinux.org/title/Installation_guide)过于繁琐，现在Arch Linux安装包提供一个类GUI的脚本[archinstall](https://wiki.archlinux.org/title/Archinstall)，按需求配置即可，可以一键处理包括KDE/GNOME/I3W等桌面在内的全部安装过程。

参考 [使用 archinstall 自动化脚本安装 Arch Linux 完整指南](https://www.linuxmi.com/archinstall-auto-arch-linux.html)，[使用 archinstall 安装 Arch Linux 和 KDE 桌面环境](https://u.sb/archlinux-archinstall/)。

我的配置如下。Profile选择安装desktop Kde, Network选择Use NetworkManager后，Kde Plasma被自动安装，不需要按参考文章手动安装桌面：

![image-20230712164243209](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202307121642425.png)

## 配置主题

kde主题应用直接安装主题基本会失败，需要去[KDE store](https://store.kde.org/browse/)下载主题并按主题的README配置

一个风格统一的主题包含几个部分：Global theme，包括桌面(desktop)，图标(icon)，鼠标(cursor)，壁纸(wallpaper)，也可以包含登录界面(sddm)和启动界面(GRUB)

(1)配置桌面

下面配置MacOS风格的 [WhiteSur Dark](https://store.kde.org/p/1400424/)，建议在Github下载完整主题:[WhiteSur-kde](https://github.com/vinceliuice/WhiteSur-kde)

使用`./install.sh`安装主题，其内部操作就是拷贝各部分配置文件到系统配置目录：

```
cp -r ${SRC_DIR}/aurorae/normal/${name}${color}* ${AURORAE_DIR}
cp -r ${SRC_DIR}/Kvantum/${name} ${KVANTUM_DIR}
cp -r ${SRC_DIR}/wallpaper/${name}* ${WALLPAPER_DIR}
cp -r ${SRC_DIR}/latte-dock/* ${LATTE_DIR}
cp -r ${SRC_DIR}/color-schemes/* ${SCHEMES_DIR}
cp -r ${SRC_DIR}/plasma/desktoptheme/${name}${pcolor} ${PLASMA_DIR}
cp -r ${SRC_DIR}/plasma/desktoptheme/icons ${PLASMA_DIR}/${name}${pcolor}
...
```

其中的latte-dock只是配置文件，需要先安装latte-dock

```
yay -S latte-dock
```

主题安装完毕后，在系统System Setting -> Appearance -> Global Theme的子目录apply各模块

(2)配置登录界面

登录界面SDDM需要独立安装，在WhiteSur-kde的sddm目录运行`install.sh`安装sddm，之后可在系统System Setting -> Startup and Shutdown中设置sddm为WhiteSur。

(3)配置启动界面

Arch linux默认没有GRUB，需要[安装GRUB](https://wiki.archlinux.org/title/GRUB)

```
sudo pacman -S grub efibootmgr
sudo grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

以 [Dark Matter GRUB Theme](https://store.kde.org/p/1603282)为例，按github的install guide安装即可。

GRUB配置文件位于/etc/default/grub，修改后使用update-grub生效。

## 中文显示和中文输入法

打开网页有中文乱码（方框），需要安装noto-fonts相关字体包

```
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji
```

安装中文输入法框架（包含pinyin输入法）并配置fcitx5，具体含义参考 [Input method](https://wiki.archlinux.org/title/Input_method)

```
sudo pacman -S fcitx5-im fcitx5-chinese-addons  fcitx5-rime
```
在desktop environment配置fcitx:
```
sudo vim /etc/environment
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

重启，按 `Win` 键搜索 `Input Method`, 点击 `Add Input Method...`搜索 `pinyin` 然后添加，按 `Ctrl` + `空格`可切换输入法

## 安装yay以使用AUR

[yay](https://aur.archlinux.org/packages/yay): Yet another yogurt. Pacman wrapper and AUR helper written in go.

yay的安装参考: [How to Install yay AUR Helper in Arch Linux [Beginner’s Guide]](https://www.debugpoint.com/install-yay-arch/)

```
sudo pacman -S base-devel git
cd /opt
sudo git clone https://aur.archlinux.org/yay.git
cd yay
sudo chown -R 用户名:users .   #必须修改yay目录的owner, yay不能被sudo编译
makepkg -si  #编译yay
```

## Go语言包换源

安装yay时makepkg会显示go包安装timeout, 需要换国内源 [goprixy.cn](https://goproxy.cn/)：

```
#临时生效
$ export GO111MODULE=on
$ export GOPROXY=https://goproxy.cn

#永久生效
$ echo "export GO111MODULE=on" >> ~/.profile
$ echo "export GOPROXY=https://goproxy.cn" >> ~/.profile
$ source ~/.profile
```

## pacman/yay换源

在archinstall时如果Mirror region选择China，则默认使用官方提供的China源，见/etc/pacman.conf的[core/extra]字段都版本了/etc/pacman.d/mirrorlist. 

但官方源速度有时很慢，建议手动添加[archlinuxcn源](https://mirrors.tuna.tsinghua.edu.cn/help/archlinuxcn/)，如果包在国内源有的话，yay速度直接起飞

有网上建议生成aur配置文件换国内源：`yay --aururl “https://aur.tuna.tsinghua.edu.cn” --save` 此处不建议，如果国内源没有的包将无法下载；换回官方源：`yay --aururl "https://aur.archlinux.org" --save`，并删除yay源配置文件`~/.config/yay/config.json`

## yay常用命令

从仓库和 AUR 中交互式搜索和安装软件包：

```
yay {{软件包|搜索词}}
```

同步并更新所有来自仓库和 AUR 的软件包：

```
yay
```

从仓库和 AUR 中安装一个新的软件包。

```
yay -S {{软件包}}
yay -Sy {{软件包}} #默认yes
```

从仓库和 AUR 中搜索软件包数据库中的关键词：

```
yay -Ss {{关键词}}
```

显示已安装软件包和系统健康状况的统计数据：

```
yay -Ps
```
卸载包：

```
yay -R 
```

## pacman更新系统

更新所有安装包：

```
sudo pacman -Syu #仅更新
sudo pacman -Syyu #如果系统有损坏包，能覆盖下载
```

## Host DNS设置

NetworkManager会自动配置DNS域名解析文件/etc/resolv.conf，且手动修改的内容每次重启会被NetworkManager覆盖。

如果要手动配置，参考 [/etc/resolv.conf](https://wiki.archlinux.org/title/NetworkManager)，设置dns.conf:

```
/etc/NetworkManager/conf.d/dns.conf
[main]
dns=none
```

## KDE Discover显示unable to load applications

```
sudo pacman -S packagekit-qt5
```

