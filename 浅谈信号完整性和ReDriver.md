---
title: 浅谈信号完整性和ReDriver
date: 2023-04-10 10:51:25
tags: ReDriver
categories: IC
---

## 信号完整性
在讨论ReDriver之前，先说明信号完整性（Signal Integrity, SI）的相关背景。
电子信号在传输过程中(无线或有线)都会受到环境噪声干扰，信号功率也会随着传输距离衰减(signal attenuation)。
通信系统中用信噪比表达的信号的好坏:

```
信噪比(dB)=10*log（信号/噪音）
```

- 当信噪比大于设备接收灵敏度时，信号能被正常接收和解析（成逻辑0/1）
- 当信噪比小于设备接收灵敏度时，信号被错误解析（错误的逻辑0/1）或者是根本解析不出信号(噪声完全淹没信号，接收端恒为0或1，没有信号变化)。

信号完整性（Signal Integrity, SI）一般指PCB电路中的电压信号的信噪比好坏。如果电路中信号能够以要求的时序、持续时间和电压幅度到达接收器，则该电路具有较好的信号完整性。反之当信号不能正常响应时，就出现了信号完整性问题。一般通过眼图观测信号完整性好坏。

信号完整性在高速电路更容易出问题，表现为信号有传输延迟和时序错误、电路串扰（电容性、电感性串扰）等。

高速信号的PCB电路设计和信号完整性密切相关，例如下图是PCB使用FR4材料和Megtron6材料，信号-频率函数显示衰减度不同。

![Attenuation versus Frequency as a function of PCB material](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202304101118830.png)

## ReDriver

Redriver能减弱信号在远距离、高噪声环境的传输中的信号完整性问题对接收端的影响。

Redriver类似通信系统中的基站，其接收传输线路中的信号，重新生成原始信号，再转发给远端设备；其输出信号基本和原始信号完全一致以保证接收端能正常解析信号。

(1)PCIe redriver

以典型的高速信号PCIe接口为例，其使用Redriver和Retimer提高信号完整性，参考：[Choosing the Right Redriver or Retimer Device to Extend PCIe Protocol Signal Range](https://www.allaboutcircuits.com/industry-articles/choosing-the-right-redriver-or-retimer-device-to-extend-pcie-protocol-signal-range/)

其RX, EQ接收PCIe信号源的TX, EQ信号，redrive生成原始信号后再从TX, EQ发送给接收端。

![Single lane redriver block diagram](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202304101126665.png)

(2)USB redriver

以多层子设备结构的USB接口为例，其使用Redriver提高子USB host的驱动能力，参考 [信号完整性 - ReDriver/ 信号中继器 / 调节器](https://www.diodes.com/zh/products/connectivity-and-timing/redrivers-repeaters/)

![redrivers application2](https://www.diodes.com/assets/Uploads/redrivers-application2__ResizedImageWzYwMCwzNTFd.png)

(3)SD redriver

即使是较低速的SD接口(MB/s级别)也有PCB设计和传输距离引起的信号完整性问题，也需要redriver解决。

如下SD redriver接收SD host的几个信号并重新生成：SD clock, SD cmd, SD data, Vdd power。

![image-20230410113046013](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202304101130119.png)
