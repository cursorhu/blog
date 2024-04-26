---
title: Ubuntu Samba配置笔记
date: 2023-01-30 10:55:24
tags: samba
categories: linux

---

## Samba基本概念

Samba是SMB protocol的应用程序实现，分为服务端和客户端；

Samba通常的使用场景：在同一局域网内的的Linux主机安装Samba服务，windows主机可以访问Linux Samba服务指定的共享目录。

在嵌入式开发中通常在windows 上编辑Samba共享目录下的代码，通过 Linux环境编译代码，而无需在两个主机间拷贝代码文件。

## Ubuntu安装Samba服务

Ubuntu 20.04和22.04 版本，安装Samba服务参考：

[www.how2shout.com/linux/how-to-install-samba-on-ubuntu-22-04-lts-jammy-linux](https://linux.how2shout.com/how-to-install-samba-on-ubuntu-22-04-lts-jammy-linux/#:~:text=Steps%20to%20install%20SAMBA%20on%20Ubuntu%2022.04%20LTS,...%206%206.%20Access%20the%20shared%20folder%20)
主要流程：

```
#install and run samba service
sudo apt install samba -y

#enable auto start samba service
sudo systemctl enable --now smbd

#firewall allow samba
sudo ufw allow samba

#add system user to sambashare group
sudo usermod -aG sambashare $USER

#set passwd for sambashare
sudo smbpasswd -a $USER

#check samba service is running
systemctl status smbd

#share the folder in ubuntu GUI checkbox
右键要共享的home文件夹properties -> local Network Share -> share this folder ->share name不能直接用用户名，可以用'用户名+Home'
```

显示无权共享：非root用户要共享/home，需要修改smb.conf:

```
sudo vim /etc/samba/smb.conf
在[global]新增usershare owner only = false
sudo systemctl restart smbd
```



## Windows访问Samba共享目录

windows下可以在文件浏览器直接访问Linux主机ip查看共享的Linux目录

![image-20230130110305978](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202301301103020.png)

首次windows访问共享目录有权限问题（不能写入），需要在Linux修改共享目录/home的权限：

```
sudo chmod 777 /home -R 
```

为了以后方便连接，可以创建网络位置，参考：[6. Access the shared folder On Windows 11 or 10](www.how2shout.com/linux/how-to-install-samba-on-ubuntu-22-04-lts-jammy-linux)

如果一个主机有两个samba共享目录，windows不允许多重连接；

要更改连接目录，操作如下：

win10系统在搜索框搜索【凭据管理器】，然后删除已有的windows samba网络连接凭据

`win+R` CMD输入 `net use * /del /y`断开所有远程链接，包括samba网络连接

重新配置windows samba网络连接

## 重装Samba

Samba的配置文件位于/etc/samba/smb.conf，如果此文件被错误配置或者误删除，需要重装Samba，流程如下:

```
sudo apt-get remove samba --purge //删掉samba服务
sudo apt-get remove samba-common --purge //这一步是关键，只重装samba不会恢复smb.conf
sudo apt-get autoremove //删掉其他samba依赖库
sudo apt-get install samba //重装，包括samba和samba-common等
```

## Samba使用示例

Samba最重要的特性是两个主机之间直接共享目录，不需要用户去拷贝文件。

在代码开发中，在windows主机的VSCode或其他编辑器直接打开Linux主机共享目录的代码，然后SSH远程Linux主机去编译。
