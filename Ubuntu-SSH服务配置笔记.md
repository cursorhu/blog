---
title: Ubuntu SSH服务配置笔记
date: 2023-02-02 11:01:24
tags: ssh
categories: linux
---

# SSH配置

## SSH基本概念

- SSH是Secure Shell缩写，实现安全远程登录

​	SSH的安全性好，原因是其对数据进行加密，方式主要有两种：对称加密（密钥加密）和非对称加密（公钥加密）
​	对称加密指加密解密使用的是同一套秘钥。Client端把密钥加密后发送给Server端，Server用同一套密钥解密。对称加密的加密强度比较高，很难破解。但是如果有一个Client端的密钥泄漏，那么整个系统的安全性就存在严重的漏洞。
​	为了解决对称加密的漏洞，于是就产生了非对称加密。
​	非对称加密有两个密钥：“公钥”和“私钥”。公钥加密后的密文，只能通过对应的私钥进行解密。想从公钥推理出私钥几乎不可能，所以非对称加密的安全性比较高。

- SSH的加密原理中，使用了RSA非对称加密算法。

​	整个过程：

​	（1）远程主机收到用户的登录请求，把自己的公钥发给用户。

​	（2）用户使用这个公钥，将登录密码加密后，发送回来。

​	（3）远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

- 关于中间人攻击（Man-in-the-middle attack）

​	中间人攻击的概念：如果有人冒充远程主机将伪造的公钥发给用户，用户很难辨别公钥真伪，用户	会和伪造主机通信而不是真正的主机。

​	因为SSH不像https协议，SSH协议的公钥是没有证书中心（CA）公证的，是主机和用户之间自己签	发的，所有SSH从原理上无法彻底防止中间人攻击

- SSH使用首次验证方式减少中间人攻击的概率：

  SSH首次连接会下载服务端的公钥，用户确认后公钥将被保存并信任。

  下次访问时客户端将会核对服务端发来的公钥和本地保存的是否相同，如果不同就发出中间人攻击的警告拒绝连接，除非用户手动清除已保存的公钥。

  所以，只要SSH首次连接没有中间人攻击，之后的SSH连接就无需担心中间人攻击



## Ubuntu安装SSH服务

```
sudo apt install ssh -y
sudo service ssh start
sudo service ssh status
```

在系统重启后ssh service会自启动，不需要`systemctl enable`去配置自启动

## Windows访问SSH服务

- 使用win+R CMD验证SSH连接

  ```
  ssh 远程主机用户名@远程主机IP
  ```

- 使用putty，xshell等工具访问主机

参考：[how2shout.com/how-to/how-to-login-into-ubuntu-using-ssh-from-windows-10-8-7](https://www.how2shout.com/how-to/how-to-login-into-ubuntu-using-ssh-from-windows-10-8-7.html#:~:text=How%20do%20I%20SSH%20into%20Ubuntu%20from%20Windows%3F,to%20Ubuntu%20server%20via%20Putty%20SSH%20client%20)

首次登陆会验证RSA公钥（1024位）的MD5 fingerprint（128位）

> ```text
> RSA key fingerprint is 98:2e:d7:e0:de:9f:ac:67:28:c2:42:2d:37:16:58:4d.
> Are you sure you want to continue connecting (yes/no)?
> ```

当首次连接的公钥被接受以后，会保存在本地文件。下次再连接这台主机会跳过公钥警告，直接提示输入密码。如果以后的连接是中间人攻击，其公钥和本地的首次公钥不同，从而保证安全性。

- 使用xftp, filezilla工具传输文件

和putty，xshell配置类似

- 使用scp命令传输文件

在linux主机之间可以用scp传输文件和目录

```
#从远程cp到本地
scp username@ip_address:/home/username/filename .
#从本地cp到远程
scp filename username@ip_address:/home/username
#拷贝目录
scp -r source_dir username@ip_address:/home/username/target_dir
```

## SSH远程开发

示例一：在SSH server和客户端建立后，可以使用VSCode和source insight等代码编辑工具改代码，用xftp传输代码到SSH Linux主机，用xshell远程编译。

示例二：VSCode安装SSH远程开发插件，可以直接远程SSH Linux主机完成代码编辑、编译，[visualstudio.com/Remote Development using SSH](https://code.visualstudio.com/docs/remote/ssh)

VSCode安装Remote-SSH插件后，添加远程host：

```
In VS Code, select **Remote-SSH: Connect to Host...** from the Command Palette (F1, Ctrl+Shift+P) and use the same `user@hostname`
```



# 远程连接相关的Ubuntu配置

## Ubuntu设置静态IP

在使用SSH和Samba连远程Ubuntu PC时，发现IP有时候会改变，因此需要配置Ubuntu PC为静态IP

1.ifconfig查看ethernet接口和当前IP

```
ubuntu@ubuntu-Z390-GAMING-X:~$ ifconfig
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.52.4.71  netmask 255.255.255.0  broadcast 10.52.4.255
```

2.编辑Ubuntu的netplan配置文件/etc/netplan/*.yaml，用tab补全找到具体的yaml，制定静态IP和DNS

参考 [Netplan network configuration tutorial for beginners](https://linuxconfig.org/netplan-network-configuration-tutorial-for-beginners)

```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eno1:
      addresses:
        - 10.52.4.71/24
      nameservers:
         addresses:
            - 10.52.1.1
            - 10.52.1.2
      #gateway4: 10.52.0.1
      routes:
         - to: default
           via: 10.52.0.1
```

以上IP和nameservers(DNS)是必须的，gateway4是网关，在ubuntu22被废弃（ubuntu22显示 ``gateway4` has been deprecated, use default routes instead.`）使用routes配置网关，参考 [netplan-gateway-has-been-deprecated](https://askubuntu.com/questions/1410750/netplan-gateway-has-been-deprecated)。怎么获取这三个值，参考以下网络命令：

```
#查看所有
nmcli
#查看gateway
netstat -rn 或 route -n
#DNS配置文件
cat /etc/resolv.conf
```

例如nmcli输出如下，其中 ` inet4 10.52.4.71/24`即当前IP/mask， `route4 default via 10.52.0.1`即默认网关，`DNS configuration servers: 10.52.1.1 10.52.1.2`即nameservers

> eno1: connected to netplan-eno1
>         "Intel I219-V"
>         ethernet (e1000e), 18:C0:4D:1F:BA:B7, hw, mtu 1500
>         ip4 default
>         inet4 10.52.4.71/24
>         route4 10.52.4.0/24 metric 100
>         route4 10.52.0.1/32 metric 100
>         route4 default via 10.52.0.1 metric 100
>         inet6 fe80::1ac0:4dff:fe1f:bab7/64
>         route6 fe80::/64 metric 256
>
> virbr0: connected (externally) to virbr0
>         "virbr0"
>         bridge, 52:54:00:13:EB:68, sw, mtu 1500
>         inet4 192.168.122.1/24
>         route4 169.254.0.0/16 metric 1000
>         route4 192.168.122.0/24 metric 0
>
> lo: unmanaged
>         "lo"
>         loopback (unknown), 00:00:00:00:00:00, sw, mtu 65536
>
> DNS configuration:
>         servers: 10.52.1.1 10.52.1.2
>         interface: eno1

3.生效

```
sudo netplan apply
```

ping确认网络正常：

> ubuntu@ubuntu-Z390-GAMING-X:~$ ping www.bing.com
> PING china.bing123.com (202.89.233.101) 56(84) bytes of data.
> 64 bytes from 202.89.233.101 (202.89.233.101): icmp_seq=1 ttl=117 time=27.1 ms
> 64 bytes from 202.89.233.101 (202.89.233.101): icmp_seq=2 ttl=117 time=27.2 ms
> 64 bytes from 202.89.233.101 (202.89.233.101): icmp_seq=3 ttl=117 time=27.2 ms

如果DNS server或gateway不符合当前网络状况，ping会失败，输出：

> Name or service not known

## Ubuntu禁止自动登出

自动登出会使SSH断开链接，按如下禁用

> setting->Privacy->Screen->Automatic Screen Lock (OFF)

