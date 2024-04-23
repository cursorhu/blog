---
title: Debian12+Gnome配置笔记
date: 2023-08-11 10:57:59
tags: debian
categories: linux

---



### 设置sudo
普通用户(以username为例)并没有被加入sudo用户组，不能使用sudo命令

```
username@debian:~$ sudo apt install xxx
username is not in the sudoers file.
```

有两种方式：

1. 在su下用visudo(nano也可以)编辑/etc/sudoers文件，新增username使sudo能获取root权限。

   ```
   # User privilege specification
   root    ALL=(ALL:ALL) ALL
   username ALL=(ALL:ALL) ALL
   ```

2. 在su下用usermod -aG sudo username，将username添加到sudo组，由于/etc/sudoers的如下设置已经将sudo组设为root权限，所以这个操作等效于使username能用sudo获取root权限。

   ```
   %sudo ALL=(ALL:ALL) ALL
   ```

- sbin的命令not found问题：执行visudo或usermod时，发现command not found. Debug过程如下：

​		使用whereis visudo查看路径是/usr/sbin；echo $PATH没有/usr/sbin，因此是环境变量问题。

   方式一：在username的~/.bashrc下添加sbin到PATH，生效之后sbin目录的命    令才可执行：

   ```
   nano .bashrc
   export PATH=$PATH:/usr/sbin
   source ~/.bashrc
   echo $PATH
   ```

   方式二：另外一种方式是加软链接，需要一个个添加

   ```
   ln -s /usr/sbin/visudo /usr/bin/visudo
   ```

- 如何退出su：使用exit，或su username，切换到username用户

### 设置apt源

- 使用iso安装的debian，首先需要取消从iso安装软件的选项：

​		software&update -> Other Software -> 取消勾选cdrom

- 再选择国内源, 例如China-> mirrors.163.com/debian，并选中main/free/non-free各种下载选项。
- 如果refresh cache界面卡死，使用`apt update`手动更新源。

### 设置gnome shell

gnome shell即gnome的桌面

(1)通过extensions自定义桌面插件

参考：[How to Use GNOME Shell Extensions [Complete Guide]](https://itsfoss.com/gnome-shell-extensions/#:~:text=Installing%20GNOME%20Shell%20Extensions%201%20Use%20gnome-shell-extensions%20package,3%20Install%20GNOME%20Shell%20Extensions%20manually.%20See%20More.)

可以通过gnome-shell-extension-manager直接安装extension，无需到网站下载

```
sudo apt install gnome-shell-extension-manager
```

推荐安装Dash to Dock和Arc Menu

(2)修改主题(theme)

参考 [How to Install and Change GNOME Theme in Linux](https://itsfoss.com/install-switch-themes-gnome-shell/)

打开extensions的User Themes插件后才可以安装自定义themes

手动安装单个GTK主题的方法：

```
#创建放置themes的目录
mkdir ~/.themes
#从gnome-look下载themes,放到.themes目录，Ctrl+H打开隐藏目录
https://www.gnome-look.org/browse/
#打开gnome tweak(默认已安装)，Appearance -> Shell -> 使用themes
```

注意

- 要完整桌面主题效果，需要安装gtk-theme，icon-theme，cursor-theme等多类theme。例如WhileSur，要达到和MacOS一样的效果，需要去GTK3/4 themes, Full Icon Themes，Cursors分别下载top5的WhileSur主题。一个主题可能有多种模式，按github install.sh能完整安装各种模式，手动下载只能安装一种模式。
- 注意gnome桌面是基于GTK，GTK is a multi-platform toolkit for creating graphical user interfaces. 所以是在gtk-theme而不是gnome-shell找。
- 要自定义登录界面的主题，去GDM themes找top5(改GDM需要先安装loginized)；要自定义GRUB主题，去GRUB themes找。
- Top 10 themes: [best-gtk-themes](https://itsfoss.com/best-gtk-themes/)
