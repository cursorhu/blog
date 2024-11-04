# SD Host Retimer Software Architecture

## 信号完整性与Redriver/Retimer

### 信号完整性(Signal Integrity, SI)

（1）数字信号与眼图

简单地讲，数字信号的逻辑0、1是依赖于模拟信号的高低电平

如下图，CMOS电平下，3.3V~2V为逻辑1，0.8V~0V(GND)为逻辑0；TTL电平下是5V~2V为逻辑1，0.8V~0V(GND)为逻辑0。

![image-20241031110053224](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202410311100267.png)

考虑到模拟信号的跳变是有个过程的，如下图，在模拟电平还没达到数字逻辑0、1对应的电压时（下图Rise time和fall time的阶段），此时采样的数据是无效的。只有模拟电平达到逻辑0、1对应的电压时（下图Bit period），才能对信号采样，得到正确的逻辑0、1。

下图即眼图，能得到正确的逻辑0、1的采样区间越大，信号质量越好。

![image-20241031110611736](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202410311106844.png)

（2）信号完整性

信号在传输过程中会参杂噪声信号，导致信号失真，眼图变差。信号失真太大，就会导致无法正确接收电路中传输的0或1信号，并导致错误的二进制值。

信号完整性即表示传输过程中的信号还保有多少原始信号的质量。

参考：[ansys.com: 什么是信号完整性？](https://www.ansys.com/zh-cn/simulation-topics/what-is-signal-integrity#what-is)

（3）高速信号的信号完整性

高速信号一般有两个特点：

1. 时钟周期短，对应时钟和数据采样频率高，即模拟信号跳变频繁，可采样时间更短，对传输线路的信号完整性要求高。
2. 信号电压水平低，比如逻辑1在TTL是5V，CMOS是3.3V，更高速数字电路常用1.8V，1.2V作为高电平。低电压能减少模拟电路的电平建立时间，才能支持更短的信号周期；此外低电平也为了信号传输的功耗更低。信号电压水平低更容易收到噪声电平干扰（更小的噪声电压就能淹没信号电压），因此对传输线路的信号完整性要求高。

因此信号完整性问题一般只在高速信号场景中需要重点考虑，低速信号一般没有信号完整性问题。

### 提高信号完整性：Redriver和Retimer

既然信号质量变差是原始信号被传输过程的噪声干扰导致，有两种思路提高信号质量：

1. 改良派：通过模拟器件滤波过滤噪声，效果有限
2. 改革派：直接重建信号，更彻底

以上两种思路落地的方案分别是redriver和retimer

参考[PCI Express® Retimers vs. Redrivers: An Eye-Popping Difference](https://www.asteralabs.com/pci-express-retimers-vs-redrivers-an-eye-popping-difference/)

- **Redriver**: A non-protocol-aware software-transparent extension device.
- **Retimer**: A physical layer protocol-aware, software-transparent extension device that forms two separate electrical link segments.

Retimer和Redriver的主要区别在于：

A retimer is a mixed signal analog/digital device that is protocol-aware and has the ability to fully recover the data, extract the embedded clock and retransmit a fresh copy of the data using a clean clock. 

1. retimer IC是包含模拟、数字的混合器件
2. retimer IC涉及到数据传输协议（例如**PCIe protocol**）：协议包含数据和时钟，retimer能接收原始数据信号再重新生成数据信号，能接收原始时钟信号再重新生成时钟信号，相当于retimer能完整地重建信号，相比redriver仅用模拟器件过滤噪声效果更好。

![image-20241031104940359](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202410311049454.png)