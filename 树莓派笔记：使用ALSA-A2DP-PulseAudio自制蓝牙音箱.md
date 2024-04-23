---
title: 树莓派笔记：使用ALSA+A2DP+PulseAudio自制蓝牙音箱
date: 2021-09-09 19:54:39
tags: raspi
categories: raspi

---

# 背景

树莓派4B自带蓝牙和Wifi, 无需外接 USB dongle；
蓝牙最常见的应用是近距离传输数据，比如蓝牙传文件，蓝牙音箱等。正好家里有个普通的usb供电的便携音箱；

本文用树莓派蓝牙+普通音箱，实现简单的蓝牙音箱。

首先需要了解Linux音频系统的整体框架：
![image-20221208194352559](C:\Users\thomas.hu\AppData\Roaming\Typora\typora-user-images\image-20221208194352559.png)

大致分为三个部分：

 - kernel/driver层的ALSA驱动框架
 - 蓝牙音频协议栈：A2DP, 这是使蓝牙具有传输音频流能力的基石; Linux官方的bluez包实现了A2DP
 - 音频应用层, Linux最常用的音频服务器是Pulse Audio

怎样理解这三层：可以类比Linux网络层：
ALSA 类似网络驱动框架
A2DP 类似TCP/UDP层
PulseAudio 类似HTTP层的服务器，类比Apache

而蓝牙连接类似http连接和会话；
声卡(输入、输出)类似网卡(Ethernet和wifi)，音频设备(音箱，麦克风)类似具体的网口设备

深入了解 ALSA 音频驱动和 A2DP 蓝牙音频协议，参考：
[Advanced Linux Sound Architecture (ALSA) project homepage](https://www.alsa-project.org/wiki/Main_Page)
[A2DP Spec](http://www.dslreports.com/r0/download/2285126~a70eb148e16b921dc323dbb977d4b4b1/A2DP_SPEC.pdf)

本文的环境
树莓派4B, 系统: ubuntu-server raspberry pi版本
音箱：usb供电，音频线
安卓手机：用于配对树莓派的蓝牙音频服务

连接示意图

     Audio source (i.e. smartphone) 
                    |
                    v
     (((  Wireless Bluetooth Channel  )))
                    |
                    v
      Raspberry PI (with A2DP service)
                    |
                    v
             Audio Interface
                    |
                    v
                 Speakers

# 使用alsa-utils测试音频设备
首先测试Linux上如何使用普通音箱
将音箱USB连到树莓派USB, 音频线连到音频接口

## 查看音频设备

ALSA在应用层提供了alsa-utils包，其含有arecord、aplay等工具来查看和使用音频设备。

    apt-get install alsa-utils

查看声卡列表：

    cat /proc/asound/cards

可以看到当前有两张声卡

card 0是树莓派的bcm2835集成声卡，card 1 是另外接的USB麦克风

注意区分声卡和音频设备，一个声卡可以管理多个音频设备，类似于"总线"和"设备"的关系。

音频设备可以细分为输入和输出两种：例如音箱是播放音频，属于输出；麦克风是录入音频，属于输入。下面分别查看这两类设备。

查看音频输入设备：

    arecord -l

查看音频输出设备：

    aplay -l

## 使用音频设备

(1)测试音频输出：

    aplay test.wav -D plughw:CARD=0,DEV=0

音频设备用 CARD 和 DEV 指定，来自于前文`aplay -l`查看音频设备的输出
测试音频(wav格式)可以在此下载：[ape8.cn](https://www.ape8.cn/wav/)

(2)测试音频输入：

使用arecord录制音频输入
-f 录制音频格式。例如 cd 表示 (16 bit little endian, 44100, stereo)
-d 录制时间，单位秒
-c 输入通道的个数，如果是麦克风阵列可能有多通道
-D 使用的设备：-D hw:1,0 表示使用 card 1 下的device 0设备

测试如下：

    arecord -f cd -d 5 -c 1 -D hw:1,0 > test.pcm

然后播放此音频：

    aplay test.pcm

# 蓝牙服务相关配置
## 蓝牙协议栈和服务的安装

首先确保系统软件是最新：

    sudo apt-get update
    sudo apt-get upgrade

安装 bluez，pulseaudio 等蓝牙基础组件，对于树莓派还要安装pi-bluetooth

    sudo apt-get install pi-bluetooth bluez bluez-tools pulseaudio pulseaudio-module-bluetooth

bluez 是Linux官方的蓝牙协议栈，其内部实现 A2DP 蓝牙音频协议，参考[bluez.org](http://www.bluez.org/about/)

PulseAudio 是Linux音频服务器, 其最主要的作用是：
PulseAudio clients can send audio to "sinks" and receive audio from "sources"

参考[PulseAudio/About](https://www.freedesktop.org/wiki/Software/PulseAudio/About/)

简单说明下蓝牙的发送、接收的概念：
蓝牙的Source端为发送码流的端，Sink端为接收码流的端；可类比生产者和消费者模型


## 启动音频服务

PulseAudio服务需要创建用户名和用户组，示例如下：

    sudo usermod -G bluetooth -a ubuntu

启动服务器

    pulseaudio --start

## 启动蓝牙配对
蓝牙首次连接需要配对，使用 bluez 的 `bluetoothctl`工具

参考：[How to Manage Bluetooth Devices on Linux Using bluetoothctl](https://www.makeuseof.com/manage-bluetooth-linux-with-bluetoothctl/)

    bluetoothctl //进入蓝牙配置模式，会显示用户为[bluetooth]#
    [bluetooth]# list //列出树莓派的蓝牙控制器列表
    [bluetooth]# agent on //注册蓝牙代理
    [bluetooth]# default-agent //使用默认代理
    [bluetooth]# discoverable on //树莓派的蓝牙可被其他设备发现
    [bluetooth]# scan on //开始扫描可连接蓝牙设备

此后选择要连接的蓝牙设备，手机蓝牙打开，`scan on`列表找到手机的 MAC地址 进行连接配对。
手机的MAC可在设置->系统信息查看

    [bluetooth]# pair <dev> //配对设备，首次需要密码
    [bluetooth]# trust <dev> //信任该设备，此后可以自动配对无需密码
    [bluetooth]# connect <dev> //建立连接

现在可以退出 bluetoothctl模式，然后测试蓝牙音频播放：

    [bluetooth]# quit
    aplay test.wav

关于蓝牙的agent，参考[bluetoothctl - What is a bluetooth agent?](https://askubuntu.com/questions/763939/bluetoothctl-what-is-a-bluetooth-agent)

## 设置自动配对连接

为了避免每次pair都要指定设备，可以配置蓝牙打开时，自动pair上次的设备。

编辑PulseAudio配置文件 `/etc/pulse/default.pa` 

    # automatically switch to newly-connected devices
    load-module module-switch-on-connect

编辑bluez配置文件 `/etc/bluetooth/main.conf`

    [Policy]
    AutoEnable=true

系统重启后只需要重启PulseAudio服务：

    pulseaudio --start

# 调试过程

## 找不到蓝牙controller
最开始bluetoothctl list显示的蓝牙控制器列表是空的，我一度怀疑买了假的raspi-4B

原因是树莓派需要安装专门的蓝牙包 pi-bluetooth，参考[rpi-4b-bluetooth-unavailable-on-ubuntu](https://raspberrypi.stackexchange.com/questions/114586/rpi-4b-bluetooth-unavailable-on-ubuntu-20-04)

树莓派很多功能都要求系统有定制包，大多数硬件失效都是定制包未安装。

## 蓝牙连接正常，播放没声音

首先确认音频设备物理连接是否正常；

然后确认PulseAudio音频服务是否正常，检查服务状态和配置文件；

    pacmd info
    pactl info

问题仍没有解决，仔细听似乎有很小的声音，检测音量配置：

    pacmd list-sinks //找到sink设备，即音箱
    pacmd set-sink-volume <sink> <value> //设置音量，value取值 [0, 65536] 代表标准音量 0~100%

参考：[adjust max possible volume in pulseaudio](https://askubuntu.com/questions/219739/adjust-max-possible-volume-in-pulseaudio#:~:text=pactl%20set-sink-volume%200%20100%25%20Where%200%20is%20the,100%25%20to%20get%20audio%20boost%20%28200%25%20for%20example%29.)

此时播放音乐可以听到但声音极小；
检查音箱的线控音量调节，调到最大；
此时蓝牙音乐只有正常音箱大概 30% 的播放音量。

原因是树莓派的USB供电驱动能力有限，同一音箱，在PC-USB供电下30%的音量大小等同于树莓派上100%的音量大小。

自此蓝牙播放音量可以达到正常水平，需要更高音量和音质建议220V供电的音箱。

# 参考内容

[Ubuntu音频设备检测](https://www.jianshu.com/p/1b79537da86d)
[Make-RPi-bluetooth-speaker-part-1](https://www.nicolabs.net/2020/Make-RPi-bluetooth-speaker-part-1)
[actuino/bt_speaker-raspberry_pi-zero_w](https://gist.github.com/actuino/9548329d1bba6663a63886067af5e4cb)
[A2DP audio streaming using Raspberry PI](https://gist.github.com/oleq/24e09112b07464acbda1#file-a2dp-autoconnect-L17)
