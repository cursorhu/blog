---
title: esp32笔记之环境搭建
date: 2023-05-04 13:07:23
tags: esp32
categories: esp32
---

esp32是乐鑫的SOC，支持Wifi, BLE等IOT功能；官方教程：[ESP-IDF编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/get-started/index.html)

# ESP-IDF环境搭建

按官方教程在Linux ubuntu搭建ESP-IDF开发环境，有clone idf一直失败的问题

本节记录不用翻墙搭建ESP-ID环境的过程，视频参考：[Linux 如何安装 ESP-IDF ESP32 开发环境搭建](https://b23.tv/VCYbC2m)

## 版本发布、下载

https://github.com/espressif/esp-idf/releases

手动下载release版本的idf压缩包，例如下载esp-idf-v5.0.1.zip

解压到 ~/esp/esp-idf (`mv esp-idf-v5.0.1 esp-idf`)

## 安装依赖
```
sudo apt-get install git wget flex bison gperf python3 python3-venv python3-setuptools cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0
```

## 安装 ESP-IDF

安装 ESP-IDF 使用的各种工具，比如编译器、调试器、Python 包等

```
cd ~/esp/esp-idf
./install.sh esp32 #esp32 chip,用此命令即可
./install.sh all #所有esp chips
```

如果安装遇到网络问题，需要设置下载服务器：

```
cd ~/esp/esp-idf
export IDF_GITHUB_ASSETS="dl.espressif.com/github_assets"
./install.sh
```

如果遇到 Python 包安装问题则需要设置 Python 源

## 设置环境变量：

每次运行都export环境变量

```
. $HOME/esp/esp-idf/export.sh
```

或把将以下语句加入 ~/.bashrc，每次执行只需要 `get_idf`：

```
alias get_idf='. $HOME/esp/esp-idf/export.sh'
```

## 串口相关设置

[与 ESP32 创建串口连接](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/get-started/establish-serial-connection.html)

查看串口: ls /dev/tty* (esp32应该是ttyUSB0)

必须将将用户添加到 `dialout` 组，从而获许串口读写权限，否则串口无法连接

```
sudo usermod -a -G dialout $USER
```

## 编译和烧录
- 设置：idf.py menuconfig
- 编译：idf.py build
- 烧录：idf.py -p PORT 【-b BAUD】 flash
- 监视：idf.py -p PORT monitor，使用快捷键 `Ctrl+]`，退出 IDF 监视器
- 一次性执行构建、烧录和监视过程：idf.py -p PORT flash monitor

# MicroPython环境搭建

分为esp32侧的Firmware和PC侧的IDE两部分。

本文是Linux环境，windows环境参考：[Thonny+MicroPython+ESP32开发环境搭建](https://doc.itprojects.cn/0006.zhishi.esp32/02.doc/index.html#/01.dajianhuanjing)

## ESP32安装MicroPython

Micropython是在嵌入式平台上运行Python的基础库，参考：https://docs.micropython.org/en/latest/

下载和安装esp32的Micropython，参考：[Installation instructions](https://micropython.org/download/esp32/)

先擦除flash, 其中esptool.py已经被esp-idf/export.sh导出到环境变量；如果ls /dev/tty*显示有ttyUSB0，但esptool.py还找不到ttyUSB0，需要重启并用`get_idf`重新export idf，再插拔esp32就可以找到.

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 erase_flash
```

烧写支持micropython的 <esp32-firmware.bin>

```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 write_flash -z 0x1000 <esp32-firmware.bin>
```

例如我的Firmware使用的是：

**[v1.20.0 (2023-04-26) .bin](https://micropython.org/resources/firmware/esp32-20230426-v1.20.0.bin)** [[.elf\]](https://micropython.org/resources/firmware/esp32-20230426-v1.20.0.elf) [[.map\]](https://micropython.org/resources/firmware/esp32-20230426-v1.20.0.map) [[Release notes\]](https://github.com/micropython/micropython/releases/tag/v1.20.0) (latest)

## 使用VScode+Pymakr搭建Micropython开发环境

总体的安装流程参考：[MicroPython: Program ESP32/ESP8266 using VS Code and Pymakr](https://randomnerdtutorials.com/micropython-esp32-esp8266-vs-code-pymakr/)

Pymakr如何使用，参考[Pymakr Getting Started](https://github.com/pycom/pymakr-vsc/blob/next/GET_STARTED.md)

写一个LED闪烁的sample code验证开发环境

```
from machine import Pin
from time import sleep

led = Pin(2, Pin.OUT) #GPIO2, output mode

while True:
  led.value(not led.value())
  sleep(0.5)
```

LED如何控制，要根据esp32具体开发板的电路图找到LED相关的GPIO，以及配什么输入/输出模式使GPIO导通/关闭。

如下图，我的esp32 LED连接到GPIO2(IO2)，并且GPIO2输出高电平时LED导通

![image-20230504200335998](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305042003091.png)

![image-20230504200411499](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305042004848.png)

选择VSCode的Pymakr Project -> connect device -> ’sync project to device‘，上传该LED python代码到esp32上运行；右键Pymakr Project的Hard reset device以后执行python代码

![image-20230505111655320](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305051116409.png)

esp32 GPIO2的LED不停闪烁

![mmexport1683202673677](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305051156061.gif)
