---
title: 树莓派笔记：使用mjpg-streamer+Apache+SSH自制网络摄像头
date: 2021-08-23 23:38:12
tags: raspi
categories: raspi
---

# 选型

 - 为什么用树莓派4：

资料多遇到容易解决问题；
性能较强适合作为终端服务器；
自带WIFI, BT5.0，GPIO 方便拓展开发IOT相关项目；
适配系统丰富，基本PC上linux版本树莓派都有对应版本

 - 为什么用USB摄像头：

为了快速实现，Linux对USB设备支持非常好，USB设备基本都是免驱；
USB摄像头支持高分辨率，带麦克风，满足其他项目拓展应用；
当然CSI接口摄像头也有优势，同等条件下其CPU占用率比USB低；不过本地测试中CPU并不是USB摄像头性能瓶颈
关于CSI和USB 摄像头区别：[CSI摄像头 vs USB摄像头](https://blog.csdn.net/ZhaoDongyu_AK47/article/details/103981905)

 - 树莓派用什么系统：

看个人喜好，我用的ubuntu server的树莓派版本，软件源基本最新；

 - 用什么云服务器：

看个人喜好和价格；云服务器最大价值在于公网IP
我目前用的Aliyun + CentOS7 系统

系统实拍：
![image-20221208194427802](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081944910.png)

# 树莓派系统安装
准备：电源，网线，SD卡
安装步骤：

 - 1.下载ubuntu server for raspi

注意一定要下载raspi版本的镜像，普通ubuntu server版本安装完不能直接使用SSH
[Install Ubuntu on a RaspberryPi](https://ubuntu.com/download/raspberry-pi)

 - 2.Win32DiskImager写.img镜像到SD卡，作为系统盘

参考：[使用win32DiskImager为树莓派4B安装系统](https://blog.csdn.net/bhniunan/article/details/104790090)

 - 3.SSH 登陆

ubuntu server for raspi系统装机启动后，连接网线到主机局域网后就可以SSH登陆
树莓派连到主机网段路由器的LAN口，树莓派系统默认开了dhcp, 用Advanced IP Scanner扫描树莓派IP

树莓派4b：mac地址“dc-a6-32”开头
![image-20221208194446543](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081944582.png)

SSH 软件看个人喜好，putty, SecureCRT, Xshell都可以，我个人使用的SecureCRT

ubuntu server for raspi系统的SSH会话初始化如下：
新建会话-> SSH2链接-> 树莓派ip -> 账户名(默认ubuntu)
初始密码：ubuntu，登陆成功后需要重设密码。

wifi配置方式参考 [树莓派安装ubuntu server, 无显示屏和键盘](https://blog.csdn.net/weixin_42378324/article/details/114631521)

 - 4.固定树莓派IP

DHCP方式每次启动树莓派IP可能不一样，有两种方式固定IP

 - MAC绑定IP
参考[TL-WR886N路由器+树莓派绑定IP地址](https://blog.csdn.net/Echozi/article/details/104210167)
 - 手动配置固定ip
[Pi4B 树莓派 ubuntu20.04 设置固定IP地址](https://blog.csdn.net/u010169607/article/details/111316624)

# USB摄像头测试

 - 首先主机win10上验证摄usb像头功能正常

设备管理器禁用笔记本原装摄像头驱动，搜索相机-> 打开视频，视频流应该正常

 - 在树莓派上验证摄像头设备

usb摄像头设备既是usb设备又是v4l2设备，应该挂载在/dev/videoX

    ls /dev/video*
    ls /dev | grep video

![image-20221208194510165](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081945191.png)
插拔摄像头确认usb摄像头对应设备是video0

# 树莓派安装mjpg-streamer
mjpg-streamer的作用是将摄像头采集的YUV/JPEG数据，封装成流服务，其他设备可以通过http方式获取图片或视频流。
mjpg-streamer属于应用层实现流媒体服务端，其底层调用的是Linux V4L2框架接口。

安装过程：

 1. 依赖库安装

    sudo apt-get install subversion libjpeg8-dev imagemagick libv4l-dev cmake git
    
 2. 安装mjpg-streamer

    git clone https://github.com/jacksonliam/mjpg-streamer.git
    cd mjpg-streamer/mjpg-streamer-experimental/
    make all
    sudo make install
    
# 局域网测试mjpg-streamer
mjpg-streamer/mjpg-streamer-experimental目录下有测试脚本：`start.sh`
环境变量添加依赖库路径：

    export LD_LIBRARY_PATH="$(pwd)" 

运行示例：

    ./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so -w ./www" 

其YUV/MJPEG的输入使用 input_uvc.so， 输出流到 http依赖于 output_http.so，`-w ./www` 表示http客户端访问时返回www文件夹下的资源，即对应的浏览器页面。

可以自定义参数，参考：

    mjpg_streamer -i "input_uvc.so --help"

修改start.sh的自定义启动语句如下：

    ./mjpg_streamer -i "./input_uvc.so -n -f 30 -r 640x480 -d /dev/video0"  -o " ./output_http.so -w ./www"

-n 用于跳过一些ioctrl请求，我的摄像头如果不用-n，有一些ioctrl会返回错误，尽管不影响流传输功能，还是跳过。
-f 设置fps，如果有卡顿考虑降低该值
-r 分辨率，1080P摄像头可以支持到1920x1080
-d 设备名，默认/dev/video0

一般USB摄像头支持直接输出压缩后的MJPEG格式图像，有的只支持YUV格式图像；
摄像头优先使用MJPEG格式，因为不用mjpg-streamer软件边采集边做压缩，减少CPU使用

启动信息：
![image-20221208194525192](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081945245.png)

此时流服务已运行，在局域网任意设备用浏览器访问`树莓派ip:流服务端口`即可获取www目录的网页资源
192.168.0.105是我树莓派固定ip, 8080是mjpg-streamer服务默认端口
![image-20221208194549504](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081945636.png)
局域网下即使是1080p 30fps也非常流畅，看不出卡顿

# 公网服务器搭建反向代理
## 反向代理的概念
正向代理和反向代理的概念图：

![image-20221208194609572](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081946618.png)

正向代理：代理的是客户端，例如GFW禁止某用户直接访问目标服务器的8080端口，但没有禁止访问正向代理服务器，客户端访问正向代理服务器，代理服务将用户请求转发给目标服务器，实现“蛙跳式”访问。对于目标服务器来说，正向代理服务器才是其客户端，用户ip对其是不可见的。
反向代理：代理的是服务端，应用于以下场景：

 - 出于安全考虑，目标服务器不直接暴露其ip和端口，用户通过访问反向代理服务器来间接访问目标服务器
 - 保证系统稳定性：反向代理服务器可以代理多个目标服务器，当用户请求量大时作为负载均衡([负载均衡和反向代理的区别](https://blog.csdn.net/ywd1992/article/details/112858537)); 支持目标服务器作为集群管理，当某个目标服务器失效时将请求转发到其他服务器, 参考[centos7下apache2.4反向代理](https://www.cnblogs.com/jkko123/p/6426857.html)

对于本项目，树莓派的mjpg-streamer进程是真正提供流媒体服务的目标服务器，阿里云公网服务器上安装apache服务，实现反向代理。

## 安装apache服务
Apache实现http web服务器；没有apache, 客户浏览器页面没办法访问对应服务。
阿里云主机 cent-OS 7 上的安装过程：

    //安装Apache
    yum install httpd
    //设置服务器开机自动启动Apache
    systemctl enable httpd.service
    //启动Apache
    systemctl start httpd.service
    //重启
    systemctl restart httpd.service
    //停止
    systemctl stop httpd.service    

启动apache后，直接访问阿里云ip，默认端口 80 即为 apache 进程端口，得到如下页面说明服务正常
![image-20221208194622426](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081946493.png)

## 配置apache为反向代理
apache相关配置路径在/etc/httpd的几个conf目录
![image-20221208194635517](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081946576.png)

`vim /etc/httpd/conf/httpd.conf` 查看关键内容如下:

    Listen 80 //监听80端口
    Include conf.modules.d/*.conf //包含module.d目录的所有conf
    DocumentRoot "/var/www/html" //默认返回该目录的html资源
    IncludeOptional conf.d/*.conf //包含conf.d目录的所有conf

`/etc/httpd/conf.modules.d`目录下的`00-proxy.conf`是针对代理的配置项，其中有大量LoadModule加载proxy模块。
配置内容是XML格式，在此自定义反向代理，追加以下内容：

    <VirtualHost *:80>
        ProxyRequests off
        <Proxy raspi>
            Order allow,deny
            Allow from all
        </Proxy>
        ProxyPass /raspi http://127.0.0.1:9020
        ProxyPassReverse /raspi http://127.0.0.1:9020
    </VirtualHost>

含义：
`<VirtualHost *:80>` 定义一个虚拟主机，*表示任意命名，端口80
`ProxyRequests off` 关闭正向代理
`<Proxy raspi>`定义一个代理对象，可以命名为*，这里命名为raspi因为后端服务是raspi流服务
`ProxyPass` 和 `ProxyPassReverse` 内容要完全一样，`ProxyPassReverse /raspi http://127.0.0.1:9020` 表示用户访问/raspi资源实际访问的是本地（apache所在云主机）的9020端口。

注意阿里云端口要支持外部可访问，需要在控制台配置安装组，参考：[阿里云服务器开放端口教程](https://developer.aliyun.com/article/767328)

我个人的配置是直接(1~65535)全部端口打开（不推荐，有风险）
![image-20221208194652256](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081946296.png)

配置完毕重启apache服务

## 配置SSH反向隧道
树莓派的mjpg-streamer服务如何连接到阿里云的apache服务？
使用SSH连通。关于SSH，参考[SSH (Secure Shell) Home Page](https://www.ssh.com/academy/ssh)

前文的SecureCRT登陆树莓派就是使用SSH2协议，下面将树莓派的mjpg-streamer服务端口通过SSH反向隧道连接到apache的代理端口

    ssh -fN -R <阿里云apache代理端口>:<树莓派localhost>:<树莓派mjpg-streamer服务端口> <阿里云服务器用户名>@<服务器IP>

例如 `ssh -fN -R 9020:localhost:8080 root@47.100.221.149`
![image-20221208194724640](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081947690.png)

输入服务器的登录密码完成通道建立，在阿里云可以查看：
![image-20221208194733933](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081947971.png)

# 验证公网可访问 mjpg-streamer 服务

 - 1.验证树莓派到apache的视频流通道：

    1. 阿里云服务器启动apache
    2. 树莓派建立SSH反向隧道
    3. 树莓派启动mjpg-streamer
    4. 在阿里云curl访问本地的代理端口

    curl 127.0.0.1:9020/?action=stream
    

如果有大量数据输出，说明连接没问题

 - 2.验证apache到客户端浏览器的反向代理通道：

使用处于任意网络的设备的浏览器，访问：

    http://云服务器IP / Apache代理名 / ?action=stream

本文中配置对应的输入是：`47.100.221.149/raspi/?action=stream`，注意`?action=stream`不能掉，直接访问`/raspi`得到的是静态页面，跳转不到`action=stream`的页面
![image-20221208194745374](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081947462.png)

直接访问 SSH 通道的 9020 端口支持主页面访问和跳转到`action=stream`页面：
![image-20221208194754703](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081947787.png)

自此验证完毕公网可访问树莓派的视频流服务

# 性能测试与优化
实测发现mjpg-streamer启动时使用 640x480分辨率, 30fps，MJPEG格式，延迟卡顿严重

树莓派 ping 阿里云延迟很小：
![image-20221208194803656](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081948699.png)
可能是阿里云带宽不足以支撑大数据量，只能降低分辨率和帧率

我的阿里云服务器只有 3M 带宽，计算一下合适的配置：

    3 * 1M/8 = 3 * 128KB = 384KB

理论上当分辨率 640x480 = 300KB, fps 要设置为 1 才几乎无延迟

测试一： 分辨率=640x480, fps=5
结果：初始延迟在 1s 以内，之后延迟增加到几秒；

测试二： 分辨率=640x480, fps=1
结果：初始延迟在 0.5s 左右，半小时后延迟也稳定在1s以内，效果明显比 fps=5 好；
测试符合理论预期，分辨率 和 FPS 要满足带宽

延迟的测试方法：手机计时，网页视频显示，放一起拍照，时间差即视频延迟
以下显示都在1分47秒，延迟小于 1s
![image-20221208194813177](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081948279.png)

注意: 树莓派长时间运行发热较明显，需要配散热片。
