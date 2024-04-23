---
title: Gitlab笔记：CentOS7部署Gitlab服务
date: 2020-04-30 14:39:02
tags: Gitlab
categories: Git
---

# 环境
阿里云ECS, CentOS7, RAM 4G

# 安装Gitlab
1.安装ssh并配置

    #安装
    sudo yum install -y curl policycoreutils-python openssh-server
    #配置开机启动
    sudo systemctl enable sshd
    #启动服务
    sudo systemctl start sshd

2.配置防火墙

    #启动防火墙
    service firewalld start
    #添加http服务到firewalld,pemmanent表示永久生效
    sudo firewall-cmd --permanent --add-service=http
    #重启防火墙
    sudo systemctl reload firewalld

3.安装gitlab

    #下载安装脚本
    curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash
    #安装
    yum install -y gitlab-ee

4.配置gitlab

    #gitlab配置文件
    vim /etc/gitlab/gitlab.rb
    #修改以下内容为主机ip和未使用的端口，否则使用默认端口8080
    external_url 'http://47.100.221.149:9030'

5.配置生效并重启gitlab

    #配置生效，改了配置需要运行
    gitlab-ctl reconfigure
    #重启服务，没改配置直接重启
    gitlab-ctl restart

![image-20221206144455394](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061444445.png)  
似乎服务都正常启动了，实际上可能有各种问题，参考问题记录

# 问题Debug记录

按以上步骤配置好后，访问主机ip:端口，直接弹出502服务端错误
![image-20221206144542817](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061445877.png)

## 配置文件权限问题?
配置文件生效命令`gitlab-ctl reconfigure`做了以下事情：

 - 配置设置写到gitlab服务直接调用的文件

实际生效的配置文件：

    vim /var/opt/gitlab/gitlab-rails/etc/gitlab.yml
    vim /opt/gitlab/embedded/service/gitlab-rails/config/gitlab.yml

![image-20221206144606669](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061446719.png)
可见配置被自动拷贝到此处，gitlab相关服务读取的是这里的配置项

 - 生成服务相关临时文件

![image-20221206144619066](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061446110.png)

原因：gitlab服务的配置文件在reconfigure时生成于/var/log/gitlab，这个文件目录默认权限不够，有些子服务不能正常运行。

解决方法：

    chmod -R 777 /var/log/gitlab

restart服务，网页即可正常访问gitlab，如果恢复该目录755权限，重启服务会502

每次重新配置，`gitlab-ctl reconfigure`似乎会删除该目录再重新写入
![image-20221206144636079](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061446151.png)

因此每次gitlab-ctl reconfigure之后都要`chmod 777`改此目录权限

## 还有502问题?
### 检查阿里云端口
首先确保主机ip是公网能访问的，不是内网ip
其次看端口是否禁用。在阿里云ECS的安全组策略中查看端口是否允许TCP输入输出
我把所有端口（1~65535）全部打开了
![image-20221206144656411](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061446454.png)

### 检查前向端口冲突
gitlab配置文件的external_url就包含前向端口

    netstat -nlp | grep 9030 (我的gitlab前向端口)

显示的是nginx服务端口，因为gitlab默认被nginx反向代理了，说明gitlab的代理服务确实占用9030。

![image-20221206144927508](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061449545.png)

### 检查子服务的端口
注意gitlab不只有一个服务，默认配置只设置了前向端口（nginx代理端口），而子服务都没配置端口，默认用了8080，如果有其他服务已经占用8080，可能子服务起不来
例如unicorn子服务：

![image-20221206144946178](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061449220.png)

查看子服务状态

    gitlab-ctl status

![image-20221206144958510](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061449573.png)

如果本地已有8080的服务，最好杀掉或换其他端口，否则要手动配置gitlab子服务端口

    unicorn['port'] = 9032 （随便一个未使用端口）
    gitlab_workhorse['auth_backend'] = "http://localhost:9032"

![image-20221206145014236](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061450285.png)
### 检查内存资源不足

阿里云2G以下RAM可能会有因内存不足，导致服务不能正常启动。
使用swap交换分区，避免内存不足的可能性。swap分区可以把部分内存分页存储到磁盘，变相“扩大”内存

    #查看现有swap分区，若未分配大小为0
    cat /proc/swaps
    #创建swap文件，大小4G，挂载到/mnt。交换分区可以使用真正的分区，或者以文件形式的分区
    dd if=/dev/zero of=/mnt/swap bs=512 count=8388616
    #使之成为swap分区
    mkswap /mnt/swap
    #修改swap分区配置
    cat /proc/sys/vm/swappiness
    sysctl -w vm.swappiness=60
    #swap分区配置永久生效
    vim /etc/sysctl.conf
    修改vm.swappiness=60
    #启动分区
    swapon /mnt/swap
    echo “/mnt/swap swap swap defaults 0 0” >> /etc/fstab
    #停用分区
    swapoff /mnt/swap
    swapoff -a > /dev/null

启用分区后运行gitlab，发现已经有一部分数据转移到了swap文件
![image-20221206145023389](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061450437.png)

# ssh访问配置
通过ssh上传下载，需要建立ssh key

    ssh-keygen   #一路回车

若创建成功，查看生成的公钥：

    cat ~/.ssh/id_rsa.pub
    ssh-rsa AAAAB3NzaC1yXXXXXXXX

添加公钥至gitlab

![image-20221206145031977](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061450049.png)

# 初始化git项目
配置git全局用户名，邮箱

    git config --global user.name "YOUR NAME"
    git config --global user.email "YOUR EMAIL@xxx.com"

初始化git仓库
可以通过xftp上传项目文件夹到gitlab服务器，再到目录内初始化.git仓库。

    cd project_folder (项目文件夹)
    git init
    git remote add origin git@YOUR_IP:YOUR_NAME/project_folder.git
    git add .
    git commit -m "Initial commit"
    git push -u origin master

这样在网页上可看到gitlab有首次commit的文件，后续也可以顺利提交和下载。
