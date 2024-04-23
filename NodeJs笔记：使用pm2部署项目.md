---
title: NodeJs笔记：使用pm2部署项目
date: 2020-05-20 14:25:24
tags: NodeJs
categories: NodeJs
---

# 背景
nodejs服务可以用`nohup node xxx.js &`后台启动，但是实际使用发现不太稳定，使用专门的node后台服务管理工具：pm2解决此问题

# pm2特性
1、内建负载均衡（使用Node cluster 集群模块）
2、后台运行
3、0秒停机重载
4、具有Ubuntu和CentOS 的启动脚本
5、停止不稳定的进程（避免无限循环）
6、控制台检测
7、提供 HTTP API
8、远程控制和实时的接口API ( Nodejs 模块,允许和PM2进程管理器交互 )
# pm2安装

    npm install -g pm2

# pm2用法

    pm2 start app.js        //启动进程
    pm2 start app.js -i 4   // 后台运行pm2，启动4个app.js
    pm2 start app.js -i max //启动，使用所有CPU核心
    pm2 start app.js --name my-api // 命名进程
    pm2 list               // 显示所有进程状态
    pm2 monit              // 监视所有进程
    pm2 logs               //  显示所有进程日志
    pm2 stop all           // 停止所有进程
    pm2 restart all        // 重启所有进程
    pm2 reload all         // 0秒停机重载进程 (用于 NETWORKED 进程)
    pm2 stop 0             // 停止指定的进程
    pm2 restart 0          // 重启指定的进程
    pm2 startup            // 产生 init 脚本 保持进程活着
    pm2 web                // 运行健壮的 computer API endpoint 
    pm2 delete 0           // 杀死指定的进程
    pm2 delete all         // 杀死全部进程

# 使用示例

部署Github项目NodeMail，每天给指定邮箱发邮件。

后台启动main.js并监测状态：

![image-20221206142729375](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061427417.png)

![image-20221206142743066](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061427110.png)

![image-20221206142752627](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061427682.png)

![image-20221206142800236](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061428280.png)
