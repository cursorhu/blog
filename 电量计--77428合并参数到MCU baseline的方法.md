# 电量计--77428的MCU sample代码交付

## 背景

77428电量计的固件代码是固化在芯片内部ROM而不是Flash，无法下载更新。

一般客户需求是提供电量计配套的Host侧MCU sample代码，其中实现可配置参数的下载流程，不同的电池使用不同的参数建模数据，在客户MCU代码运行时下载到电量计。

可配置参数的来源：

（1）电池厂商提供的电池规格书文档和参数信息表excel，FAE和客户沟通获得。

（1）电芯特征数据：电芯拿到实验室 -> 用循环仪器测试工具对电池循环充放电 -> 得到循环仪原始采集文件csv，包括电压电流电荷 -> 使用软件TableMaker从循环仪采集文件中提取出参数文件txt（RC table.txt和OCV table.txt），这部分需要FAE，软件开发，循环仪操作人协作。

可配置参数的写入：MCU host通过I2C给77428 chip中的F/W（固化ROM）通信，写配置参数，以适配不同的电池模块。

## 电池建模

拿到客户电池，根据电池规格书的放电电流参数范围，决定要采集的电流范围；根据客户需求的温度范围，决定采集的温度范围。

77428 A2版本的固件要求电池建模必须至少采集4个电流 * 4个温度，共16种情况；每个采集得到电荷变化算出剩余容量RC

循环仪采集的建模数据集合示例：

[JG-20250321.7z](https://o2micro-my.sharepoint.com/:u:/p/thomas_hu/Ebuc3fWIJndPnko8315TasIBkeIUS6LSJDYvnNlXK7-8TQ?e=OqHlto)

经过77428 版本的TableMaker工具转换成77428 MCU sample代码能用的txt，77428项目使用的电池特征参数文件是OCV.txt和RC.txt，建模工具输出的.c/.h是给其他项目使用。

[JG TABLE-20250321.7z](https://o2micro-my.sharepoint.com/:u:/p/thomas_hu/ETi91u6I8-pFsxZtX3DQWlMB1pY2zaLarvceKMmWCYZR8g?e=FHpIXc)

![image-20250321144922884](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211449916.png)

OCV table默认用OCV/FalconLY里面的x数据（36个），用OCV目录的也可以，取y数据（36个）

![image-20250321144916301](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211449354.png)

## MCU sample代码合并电池参数的方法

（1）配OCV voltage，来源是OCV.txt的x或y。一般采样36个电压，电压范围来自规格书

![image-20250214170240774](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502141702910.png)

（2）配RC.txt中的电压，电流，温度区间，以及RC容量值。

注意RC table的成员要和77428温度数组的项数目对应。

77428 A2版本FW规定死了Host代码配置的电流是4组，温度是4组，电压是36个采样，如果Host代码定义不符合，会写入参数报错。意味着循环仪对电池的采样测试至少要做4*4 = 16组数据，可以多测但不能少。

![image-20250320173549846](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201735883.png)

![image-20250214171651607](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502141716710.png)

（3）配MCU .h中的充放电参数，数据来自FAE提供的excel（源数据来自电池规格书）

![image-20250214172136527](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502141721613.png)

注：电池建模采集电压的范围要根据具体电池的规格书确定。

![image-20250321140503084](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211405169.png)

注：DFCC table是调整算法的策略，其温度、电流、电压区间与电池建模的区间无关。没有调算法的需求就先不动。

![image-20250321140625765](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211406800.png)

## MCU sample代码分析

MCU sample代码中有下发I2C配置77428的示例，客户需要根据自己的平台按类似的I2C包格式实现代码

（1）下载电池参数到77428。

![image-20250214172824010](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502141728106.png)

（2）通过77428自定义I2C SBS命令，查询电池状态

![image-20250214173133900](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502141731968.png)

（3） SBS各命令的含义和格式要求见Firmware Specification

Host给电量计发的命令格式要按电量计specification规定，读写指定长度的数据，且带校验字节（PEC：Packet Error Check）

![image-20250321141318580](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211413628.png)

## MCU sample客户交付示例

[SW6000_SD77428_MCU_20250214.zip](https://o2micro-my.sharepoint.com/:u:/p/thomas_hu/EePUWfAMJ2BMqXaudjTf96MBcnzJImOLlWLKQ6FRuVb5Og?e=uZMr2S)

## MCU sample内部测试环境：代码移植到STM32F407运行

[2.STM32F407测试环境](https://o2micro-my.sharepoint.com/:f:/p/thomas_hu/EmPIBLrw8ylMmO33dsfI9DQBnrkCqU9n5anKoXvQ-5et2w?e=dUpdfx)

## 问题记录

TODO